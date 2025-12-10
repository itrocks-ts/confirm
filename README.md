[![npm version](https://img.shields.io/npm/v/@itrocks/confirm?logo=npm)](https://www.npmjs.org/package/@itrocks/confirm)
[![npm downloads](https://img.shields.io/npm/dm/@itrocks/confirm)](https://www.npmjs.org/package/@itrocks/confirm)
[![GitHub](https://img.shields.io/github/last-commit/itrocks-ts/confirm?color=2dba4e&label=commit&logo=github)](https://github.com/itrocks-ts/confirm)
[![issues](https://img.shields.io/github/issues/itrocks-ts/confirm)](https://github.com/itrocks-ts/confirm/issues)
[![discord](https://img.shields.io/discord/1314141024020467782?color=7289da&label=discord&logo=discord&logoColor=white)](https://25.re/ditr)

# confirm

Seamlessly adds a user confirmation step before executing an action.

*This documentation was written by an artificial intelligence and may contain errors or approximations.
It has not yet been fully reviewed by a human. If anything seems unclear or incomplete,
please feel free to contact the author of this package.*

## Installation

```bash
npm i @itrocks/confirm
```

## Usage

`@itrocks/confirm` provides an `Action` subclass that inserts a confirmation
page between the user click and the actual execution of the action.

The usual flow is:

1. In your action's `html` handler, call `confirm.confirmed(request)`.  
   If it returns `undefined`, you must display the confirmation page.  
   If it returns a `Request`, the user has already confirmed and you can
   safely continue.
2. To display the confirmation page, return `confirm.html(request, message)`.

The confirmation state is transparently stored in the session so that the
second request can resume the original action with the same parameters.

### Minimal example

```ts
import { Action }  from '@itrocks/action'
import { Request } from '@itrocks/action-request'
import { Confirm } from '@itrocks/confirm'

export class DangerousAction<T extends object = object> extends Action<T>
{

	html(request: Request<T>)
	{
		const confirm   = new Confirm<T>()
		const confirmed = confirm.confirmed(request)
		if (!confirmed) {
			return confirm.html(request, 'Do you really want to perform this action?')
		}
		request = confirmed

		// Proceed with the real work after confirmation
		return this.htmlTemplateResponse({ done: true }, request, __dirname + '/dangerous.html')
	}

}
```

### Realistic example: confirm deletion

The `@itrocks/delete` package uses `@itrocks/confirm` to prevent accidental
data loss. The pattern is reusable for any destructive or costly operation.

```ts
import { Action }     from '@itrocks/action'
import { Need }       from '@itrocks/action'
import { Request }    from '@itrocks/action-request'
import { Confirm }    from '@itrocks/confirm'
import { Route }      from '@itrocks/route'
import { dataSource } from '@itrocks/storage'
import { tr }         from '@itrocks/translate'

@Need('object')
@Route('/delete')
export class Delete<T extends object = object> extends Action<T>
{

	async html(request: Request<T>)
	{
		const confirm   = new Confirm<T>()
		const confirmed = confirm.confirmed(request)
		if (!confirmed) {
			// First step: show confirmation message
			return confirm.html(
				request,
				tr('Do you confirm deletion') + tr('?') + '\n' + tr('All data will be lost') + '.'
			)
		}
		// Second step: user confirmed, resume original request
		request = confirmed

		const objects = await request.getObjects()
		for (const object of objects) {
			await dataSource().delete(object)
		}
		return this.htmlTemplateResponse({ objects, type: request.type }, request, __dirname + '/delete.html')
	}

}
```

In the first call, `confirm.confirmed(request)` returns `undefined` and your
action returns the confirmation page. Once the user validates the form,
`confirm.confirmed(request)` returns a reconstructed `Request` identical to
the original one (including parameters and type), and your action continues
where it left off.

## API

### `class Confirm<T extends object = object> extends Action<T>`

Helper action used to add a confirmation step before executing another
`Action`.

You normally create an instance inside your own action's handler methods
(`html`, or another format if needed) and delegate to its helpers.

#### `confirmed(request: Request<T>): Request<T> | undefined`

Checks whether the current HTTP request corresponds to a previously confirmed
action.

- **Parameters**
  - `request`: the current `Request<T>` received by your action.
- **Returns**
  - `Request<T>`: a cloned `Request` representing the original (pre‑confirm)
    call, ready to be used by your action, or
  - `undefined` if this is the first step and the user has not confirmed yet.

Usage pattern:

```ts
const confirm   = new Confirm<T>()
const confirmed = confirm.confirmed(request)
if (!confirmed) {
	return confirm.html(request, 'Please confirm…')
}
request = confirmed
// continue with the confirmed action
```

#### `generateConfirmHash(request: Request<T>): string`

Generates and stores a confirmation token in the user's session for the given
request, and returns the corresponding hash.

You do not usually call this directly; it is handled for you by
`Confirm.html`. It is exposed for advanced use cases where you want to
integrate the confirmation token in a custom flow or template.

- **Parameters**
  - `request`: the original `Request<T>` you want to put on hold until
    confirmation.
- **Returns**
  - `string`: an opaque confirmation token to be sent back by the client on
    confirmation.

#### `html(request: Request<T>, message: string): Promise<HtmlResponse>`

Builds and returns an `HtmlResponse` containing a standard confirmation page.

- **Parameters**
  - `request`: the current `Request<T>`. It will be cloned and stored in the
    session until the user confirms.
  - `message`: the confirmation message to display. A single line break (`\n`)
    is converted to an HTML line break for convenience.
- **Returns**
  - `Promise<HtmlResponse>` resolving to the confirmation page.

The generated page is compatible with the standard `@itrocks/ux-core` / modal
stack. It will send back the confirmation token so that a subsequent request
can be recognised by `confirmed()`.

## Typical use cases

- Protecting destructive actions (delete, truncate, reset, archive…).
- Asking explicit confirmation before sending an operation to a downstream
  system (payment, irreversible processing, external API calls).
- Confirming bulk actions selected from a list of objects.
- Adding a lightweight confirmation modal to dangerous administrative
  features without duplicating HTML/JS logic.
