# Drive a web page in the built-in browser

Read this when: you click through a React, Angular or Blazor page with the browser pane, or a form check needs the browser.

- Override dialogs first (they reset on every navigation): `window.__alerts=[]; window.alert = m => window.__alerts.push(String(m)); window.confirm = () => true;`, and read `__alerts` after each action; that is how error paths (400 bad value, 400 in use) are seen.
- **Angular `ngModel`:** set `el.value = v; el.dispatchEvent(new Event('input',{bubbles:true})); el.dispatchEvent(new Event('change',{bubbles:true}))`; click with `.click()`; `await` about 1 s after each server call.
- **React controlled input:** use the native setter of the element's own prototype (`HTMLInputElement`, `HTMLTextAreaElement` or `HTMLSelectElement`; the wrong one throws "Illegal invocation"): `Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set.call(el, v)`, then dispatch `input` / `change`. A text field generated as a `<textarea>` looks like an input.
- **After every write, query the database** (`sqlcmd`, `psql`) to confirm the row changed; the grid can lie (updated client-side) and an un-subscribed HTTP call changes nothing. A cleared date must be `NULL`, not an empty string.
- Screenshots time out when the window is behind others; prefer `innerText`, `get_page_text`, `getBoundingClientRect()`, `getComputedStyle`, `querySelectorAll`, `fetch`. A hidden or minimized pane cannot take clicks: drive with `javascript_tool`.
- **Set the viewport first** (`resize_window` 1280x900) and reset it with the `desktop` preset; a collapsed pane makes every element measure width 0. A closed `<dialog>` or hidden tab also measures zeros: find `dialog[open]` and check ancestors for `hidden`.
- A script that sets `location.href` is cut off: use the navigate tool, then a second script. The console buffer keeps errors from earlier loads: reload before concluding an error is current. Editing an HTML file can move the pane to the local file: navigate back to the app URL, and do the browser step after the edits. The token survives `navigate` within the app; log in through the form once.
- Use test values the server's own validation accepts; a rejected value proves the error path only, so follow it with a valid one.

## Form validation in the browser

- **`form.querySelector(':invalid')` returns the `<fieldset>` around an invalid input** (a fieldset matches `:invalid` too); read `form.checkValidity()` and the input itself (`querySelectorAll('input:invalid')`) when the check names the field.
- **Test the add path with every required field of the parent**, not only the child: a blank required field elsewhere refuses the Submit, which looks like a failed save until `checkValidity()` says so.

## Blazor in the browser pane

- **Blazor in the browser pane:** set a value with the native setter and dispatch `input` and `change` (an `InputSelect` and `InputNumber` bind on `change`), click with `.click()`, wait about a second, then read the rows back from the SQLite file with Python.
