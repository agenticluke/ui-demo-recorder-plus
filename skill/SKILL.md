---
name: ui-demo
description: Record polished web app demo videos with Playwright. Use when the user asks for a UI demo, walkthrough, screen recording, product tour, or tutorial. Explore the app, test the full flow, then record a WebM video with a visible cursor and clear timing.
origin: ECC
---

# UI Demo Video Recorder

> Original skill by **ECC**. Keep this credit in copies and changed versions.

Record a clear WebM demo of a web app with Playwright. Show a visible cursor, smooth moves, slow typing, and a simple story.

Do not add tracking, data reports, or outside calls.

## Use this skill when

Use this skill when the user asks for:

- A web app demo
- A screen recording
- A product tour
- A how-to video
- A feature walkthrough
- A video for docs or new users

Do not use it for a still image or a non-web app.

## Ask for key facts

Get these facts before work starts:

- Start URL
- Steps to show
- Login needs
- Test data to use
- Screen size
- Output path
- Private data that must stay hidden

If a small detail is missing, use a safe default. Do not guess a password, payment step, or action that may harm data.

## Rules

Always use these three phases:

1. Explore
2. Rehearse
3. Record

Do not skip to recording.

Use a test account when possible. Do not record real passwords, keys, payment data, private names, or alerts from other apps.

Do not send a real form, buy an item, delete data, publish content, or email people unless the user asked for that exact action. Use a test site or stop before the last step.

## Phase 1: Explore

Open each page in the demo. Learn how the real page works before writing the script.

Check:

- Form fields and their types
- Exact button text
- Required fields
- Form error text
- Drop-down choices
- Text editors and mention tools
- Fields that appear after another step
- Tabs, dialogs, popups, and menus
- Tables and column names
- Loading states
- Toast messages
- New tabs and downloads
- Iframes and shadow roots
- Small screen layouts
- Cookie banners and help bubbles

Do not rely on the tag alone. A drop-down may be a custom button. A text box may use `contenteditable`. Two buttons may have the same text.

Prefer Playwright locators in this order:

1. `getByRole`
2. `getByLabel`
3. `getByPlaceholder`
4. Stable test ID
5. CSS selector as a last choice

Avoid selectors based on row number, long class names, or changing IDs.

### List visible controls

Run this on each page:

```javascript
const controls = await page.evaluate(() => {
  const items = [];

  document
    .querySelectorAll(
      'input, select, textarea, button, a, [contenteditable="true"], [role]'
    )
    .forEach((el) => {
      const style = getComputedStyle(el);
      const box = el.getBoundingClientRect();
      const visible =
        style.visibility !== 'hidden' &&
        style.display !== 'none' &&
        box.width > 0 &&
        box.height > 0;

      if (!visible) return;

      items.push({
        tag: el.tagName.toLowerCase(),
        type: el.getAttribute('type') || '',
        role: el.getAttribute('role') || '',
        name: el.getAttribute('name') || '',
        label: el.getAttribute('aria-label') || '',
        placeholder: el.getAttribute('placeholder') || '',
        text: (el.textContent || '').trim().slice(0, 60),
        disabled: el.matches(':disabled, [aria-disabled="true"]'),
      });
    });

  return items;
});

console.log(JSON.stringify(controls, null, 2));
```

For a native `<select>`, list its choices:

```javascript
const choices = await page.locator('select').evaluateAll((selects) =>
  selects.map((select) => ({
    name: select.name,
    options: Array.from(select.options).map((option) => ({
      value: option.value,
      text: option.text.trim(),
      disabled: option.disabled,
    })),
  }))
);

console.log(JSON.stringify(choices, null, 2));
```

Do not pick a blank choice or a prompt such as "Select." A prompt may use `value=""` or `value="0"`.

### Make a page map

Write a short map before making the demo:

```text
/orders/new
  Customer: combobox named "Customer"
  Date: input labeled "Need by"
  Note: textarea labeled "Reason"
  Add item: button named "Add item"
  Save: button named "Save order"

/orders/123
  Status: text "Draft"
  Comment: input with placeholder "Write a comment"
  Send: button named "Send", off until text is entered
```

## Phase 2: Rehearse

Run the full demo with video off.

A rehearsal must test actions, not just check that controls exist. Fill fields, open menus, move between pages, and check the result after each main step.

Use a new test record or reset the app before each run. This keeps old data from changing the demo.

### Check a control

```javascript
async function needVisible(locator, label) {
  try {
    await locator.waitFor({ state: 'visible', timeout: 5000 });
    console.log(`OK: ${label}`);
    return locator;
  } catch {
    console.error(`FAIL: ${label} is not visible`);
    throw new Error(`Rehearsal failed at: ${label}`);
  }
}
```

Use strict locators when you can:

```javascript
const saveButton = page.getByRole('button', { name: 'Save order', exact: true });
await needVisible(saveButton, 'Save order button');
```

If a locator finds more than one item, make it more exact. Do not use `.first()` until you know why the first match is right.

### Wait for page state

Do not use long fixed waits to hide a race. Wait for a real sign that the app is ready:

```javascript
await Promise.all([
  page.waitForURL('**/orders/*'),
  page.getByRole('button', { name: 'Save order' }).click(),
]);

await page.getByText('Order saved', { exact: true }).waitFor({
  state: 'visible',
});
```

Small waits are fine for demo pace. They are not a fix for a bad wait rule.

### Stop on app errors

Fail the rehearsal on page errors:

```javascript
const appErrors = [];

page.on('pageerror', (error) => {
  appErrors.push(`Page error: ${error.message}`);
});

page.on('console', (message) => {
  if (message.type() === 'error') {
    appErrors.push(`Console error: ${message.text()}`);
  }
});
```

Some apps log known safe errors. Only ignore one when the reason is clear and written in the script.

At the end:

```javascript
if (appErrors.length) {
  throw new Error(appErrors.join('\n'));
}
```

### If rehearsal fails

1. Save a screenshot.
2. Print the current URL.
3. List visible controls.
4. Fix the locator or wait rule.
5. Reset the test data.
6. Run the full rehearsal again.

Record only after the whole flow passes.

## Phase 3: Record

Use the same tested flow. Change only the pace and video settings.

### Set a stable screen

Use a fixed size. Hide browser bars by recording the page context.

```javascript
const context = await browser.newContext({
  viewport: { width: 1440, height: 900 },
  recordVideo: {
    dir: 'artifacts/video',
    size: { width: 1440, height: 900 },
  },
  colorScheme: 'light',
  reducedMotion: 'reduce',
  locale: 'en-US',
});
```

Keep the screen size, theme, language, and zoom fixed. Turn off app motion only if it does not hide a feature the user wants to show.

Wait for fonts and images before the first action:

```javascript
await page.waitForLoadState('networkidle');
await page.evaluate(() => document.fonts.ready);
```

Do not use `networkidle` if the app has a live link that never ends. In that case, wait for a known page title or main control.

### Tell a simple story

Use this order unless the user gives another one:

1. Start at a clear page.
2. Pause so the viewer knows where they are.
3. Show the main task.
4. Show one useful extra choice.
5. End on the saved result.

Keep the demo short. Remove steps that do not help the story.

### Pace

Good base times are:

- After sign-in: 4 seconds
- After a new page: 3 seconds
- Before a click: 0.4 seconds
- After a click: 1 to 2 seconds
- Between main steps: 1.5 to 2 seconds
- After the final result: 3 seconds
- Typing: 25 to 40 ms per key

Use shorter waits for simple pages. Use longer waits when the viewer must read text.

### Add a visible cursor

Use `addInitScript` so the cursor comes back after each page load:

```javascript
await context.addInitScript(() => {
  const addCursor = () => {
    if (!document.body || document.getElementById('demo-cursor')) return;

    const cursor = document.createElement('div');
    cursor.id = 'demo-cursor';
    cursor.setAttribute('aria-hidden', 'true');
    cursor.innerHTML = `
      <svg width="24" height="24" viewBox="0 0 24 24"
        xmlns="http://www.w3.org/2000/svg">
        <path d="M5 3L19 12L12 13L9 20L5 3Z"
          fill="white" stroke="black" stroke-width="1.5"
          stroke-linejoin="round"/>
      </svg>
    `;

    Object.assign(cursor.style, {
      position: 'fixed',
      left: '0px',
      top: '0px',
      width: '24px',
      height: '24px',
      zIndex: '2147483647',
      pointerEvents: 'none',
      filter: 'drop-shadow(1px 1px 2px rgba(0,0,0,0.35))',
    });

    document.body.appendChild(cursor);

    document.addEventListener(
      'mousemove',
      (event) => {
        cursor.style.transform =
          `translate(${event.clientX}px, ${event.clientY}px)`;
      },
      { passive: true }
    );
  };

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', addCursor, { once: true });
  } else {
    addCursor();
  }
});
```

This cursor does not enter cross-site iframes. If the target is inside one, use the browser cursor or add a safe overlay inside that frame when access is allowed.

### Move, then click

```javascript
async function moveAndClick(page, locator, label, options = {}) {
  const {
    before = 400,
    after = 1000,
    steps = 14,
    ...clickOptions
  } = options;

  await locator.waitFor({ state: 'visible', timeout: 5000 });
  await locator.scrollIntoViewIfNeeded();

  const box = await locator.boundingBox();
  if (!box) {
    throw new Error(`No screen box for: ${label}`);
  }

  await page.mouse.move(
    box.x + box.width / 2,
    box.y + box.height / 2,
    { steps }
  );

  await page.waitForTimeout(before);
  await locator.click(clickOptions);
  await page.waitForTimeout(after);
}
```

Do not skip a failed action. Stop the run so a broken demo is not saved as final.

### Type at a clear speed

```javascript
async function typeSlowly(page, locator, text, label, delay = 35) {
  await locator.waitFor({ state: 'visible', timeout: 5000 });
  await moveAndClick(page, locator, label, { after: 200 });

  await locator.fill('');
  await locator.pressSequentially(text, { delay });
  await page.waitForTimeout(500);
}
```

For a date, file input, code editor, or rich text box, use the method that the control needs. Check the final value after input.

Do not type secrets on screen. If sign-in must be shown, use a fake test password or start from saved login state.

### Scroll with care

Keep the target in view. Avoid fast jumps.

```javascript
await page.evaluate(() => {
  window.scrollTo({ top: 500, behavior: 'smooth' });
});
await page.waitForTimeout(1200);
```

For a control, prefer:

```javascript
await locator.scrollIntoViewIfNeeded();
```

### Show a dashboard

Move over a few key parts. Do not point at every card.

```javascript
async function panElements(page, locator, maxCount = 5) {
  const count = Math.min(await locator.count(), maxCount);

  for (let index = 0; index < count; index += 1) {
    const item = locator.nth(index);
    const box = await item.boundingBox();

    if (!box) continue;

    await page.mouse.move(
      box.x + box.width / 2,
      box.y + box.height / 2,
      { steps: 12 }
    );
    await page.waitForTimeout(700);
  }
}
```

### Handle common hard cases

- Cookie banner: close it before the story starts.
- Popup or tooltip: close it if it hides the task.
- New tab: wait for it with `context.waitForEvent('page')`.
- Download: use `page.waitForEvent('download')` and save to a test path.
- Dialog: set `page.on('dialog', ...)` before the action.
- Iframe: use `page.frameLocator(...)`.
- Shadow root: use Playwright locators. They can enter open shadow roots.
- File upload: use a safe test file.
- Canvas: use mouse points only after checking the screen size.
- Live data: freeze or seed it when the app allows this.
- Private text: hide it before recording. Do not blur it after capture as the first plan.
- Failed save: stop and fix the cause. Do not end on an error.
- Video timeout: split a long tour into short videos.

## Save and check the video

Close the page and context so Playwright finishes the file:

```javascript
const video = page.video();

await page.close();
await context.close();

const videoPath = await video.path();
console.log(`Video saved at: ${videoPath}`);
```

Then check:

- The file exists and is not empty.
- The first frame is ready.
- The cursor is easy to see.
- No secret or private data is shown.
- Text can be read.
- No click is cut off.
- The final result stays on screen.
- There are no blank frames or error pages.
- The full task shown in the video worked.

Keep failed takes apart from the final file.

## Concrete example

This example records a safe draft order. It stops after saving the draft.

```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ headless: true });

  const context = await browser.newContext({
    viewport: { width: 1440, height: 900 },
    recordVideo: {
      dir: 'artifacts/video',
      size: { width: 1440, height: 900 },
    },
    reducedMotion: 'reduce',
  });

  await context.addInitScript(() => {
    const addCursor = () => {
      if (!document.body || document.getElementById('demo-cursor')) return;

      const cursor = document.createElement('div');
      cursor.id = 'demo-cursor';
      cursor.innerHTML =
        '<div style="font-size:28px;color:white;text-shadow:0 0 2px black">➤</div>';

      Object.assign(cursor.style, {
        position: 'fixed',
        left: '0',
        top: '0',
        zIndex: '2147483647',
        pointerEvents: 'none',
      });

      document.body.appendChild(cursor);

      document.addEventListener('mousemove', (event) => {
        cursor.style.transform =
          `translate(${event.clientX}px, ${event.clientY}px)`;
      });
    };

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', addCursor, { once: true });
    } else {
      addCursor();
    }
  });

  const page = await context.newPage();
  await page.goto('http://localhost:3000/orders/new');

  const customer = page.getByLabel('Customer');
  const note = page.getByLabel('Reason');
  const save = page.getByRole('button', {
    name: 'Save draft',
    exact: true,
  });

  await needVisible(customer, 'Customer field');
  await needVisible(note, 'Reason field');
  await needVisible(save, 'Save draft button');

  await customer.selectOption({ label: 'Northwind Test' });
  await page.waitForTimeout(800);

  await typeSlowly(
    page,
    note,
    'Restock the test room.',
    'Reason field'
  );

  await moveAndClick(page, save, 'Save draft button');

  await page.getByText('Draft saved', { exact: true }).waitFor({
    state: 'visible',
  });

  await page.waitForTimeout(3000);

  const video = page.video();
  await page.close();
  await context.close();

  console.log(`Video saved at: ${await video.path()}`);
  await browser.close();
})();
```

Run this flow once without `recordVideo`. Fix all errors. Then run it again with recording on.