---
name: testing-playwright
description: Browser automation testing with Playwright for GAM login, SLO, and redirect flows
---

# Playwright Testing for GAM
Scope: reproducible browser-based verification of GAM flows when server-side traces alone are not enough

Related files:
- [GAM Debugging Core](common-debugging.md) — trace activation and diagnostic workflow
- [GAM Trace Analyzer](common-trace-analyzer.md) — trace analysis methodology
- [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md) — complementary DB-level diagnostics

---

## Why Playwright (not curl / PowerShell)
GeneXus AJAX POST requires a full browser context to function correctly:

- Session cookies: GeneXus maintains server-side session state tied to cookies
- GXState: hidden form field containing serialized page state
- AJAX_SECURITY_TOKEN: per-session anti-CSRF token validated on every AJAX POST

Without browser context:
- `curl` or `Invoke-WebRequest` sends a POST without valid GXState / AJAX_SECURITY_TOKEN
- The server returns HTTP 440 (Login Timeout / Session Expired)
- Expected behavior — AJAX security token validation fails because there is no browser session

## Installation
```bash
npm install playwright
npx playwright install chromium
```

## Script Pattern: GAM Login Test
```javascript
const { chromium } = require('playwright');

(async () => {
	const browser = await chromium.launch({ headless: true });
	const context = await browser.newContext();
	const page = await context.newPage();

	const networkLog = [];
	page.on('request', req => {
		networkLog.push({ method: req.method(), url: req.url() });
	});
	page.on('response', res => {
		networkLog.push({ status: res.status(), url: res.url() });
	});

	page.on('console', msg => {
		console.log(`[CONSOLE ${msg.type()}] ${msg.text()}`);
	});

	try {
		await page.goto('http://localhost/MyApp/login.aspx', {
			waitUntil: 'networkidle',
			timeout: 30000
		});

		await page.fill('input[name*="UserName"], input[name*="usr"]', '<admin_user>');
		await page.fill('input[name*="Password"], input[name*="pwd"]', '<admin-password>');

		const [response] = await Promise.all([
			page.waitForResponse(res =>
				res.url().includes('login') && res.request().method() === 'POST',
				{ timeout: 15000 }
			),
			page.click('input[type="button"][value*="Log"], button:has-text("Log")')
		]);

		const body = await response.text();
		console.log('Response status:', response.status());
		console.log('Response body:', body);

		if (body.includes('"redirect"')) {
			console.log('LOGIN SUCCESS — redirect command received');
		} else {
			console.log('LOGIN FAILED or UNEXPECTED RESPONSE');
		}

		await page.screenshot({ path: 'login_result.png', fullPage: true });
		await page.waitForTimeout(2000);
		console.log('Final URL:', page.url());

	} catch (error) {
		console.error('Error:', error.message);
		await page.screenshot({ path: 'login_error.png', fullPage: true });
	} finally {
		console.log('\n--- Network Log ---');
		networkLog.forEach(entry => console.log(JSON.stringify(entry)));
		await browser.close();
	}
})();
```

## Interpreting Results
- `{"gxCommands":[{"redirect":{"url":"gamhome", …}}]}` — successful login, GAM redirects to home
- `{"gxCommands":[{"redirect":{"url":"login", …}}]}` — login failed, credentials rejected, redirecting back to login
- HTTP 440 from curl — AJAX security token validation failed (expected without browser context)
- Page refreshes with no error — HTTP 440 caught by `gxgral.js` which reloads the page silently
- Redirect to gamhome but blank page — gamhome object exists but no Home Object configured for the Application

## What to Capture for Diagnosis
- Network log — all requests and responses during the login flow
- Console messages — JavaScript errors, GeneXus client-side diagnostics
- Screenshot — visual state of the page after login attempt
- Response body — the AJAX JSON response from the POST
- Final URL — where the browser ended up after the flow
