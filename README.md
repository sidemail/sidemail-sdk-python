# Sidemail Python Library

Official Sidemail.io Python library providing convenient access to the Sidemail API from Python applications.

## Requirements

- Python 3.8+

## Installation

```bash
pip install sidemail
```

## Usage

First, the package needs to be configured with your project's API key, which you can find in the [Sidemail Dashboard](https://client.sidemail.io/) after you signed up.

```python
from sidemail import Sidemail
sm = Sidemail(api_key="xxxxx") # or set SIDEMAIL_API_KEY env var

resp = sm.send_email(
  toAddress="user@email.com",
  fromAddress="you@example.com",
  fromName="Your App",
  templateName="Welcome",
  templateProps={"foo": "bar"},
)

print(f"Email sent! ID: {resp.id}")
```

The response looks like:

```json
{
  "id": "5e858953daf20f3aac50a3da",
  "status": "queued"
}
```

Shortcut `sm.send_email(...)` calls `sm.email.send(...)` under the hood.

### Authentication

Explicit key:

```python
from sidemail import Sidemail
sm = Sidemail(api_key="your-api-key")
```

Or if you set environment variable `SIDEMAIL_API_KEY`, then simply:

```python
from sidemail import Sidemail
sm = Sidemail() # reads SIDEMAIL_API_KEY
```

### Client configuration

```python
import httpx
from sidemail import Sidemail

sm = Sidemail(
	api_key="your-api-key",
	base_url="https://api.sidemail.io/v1", # override for testing/mocking
	timeout=10.0,  # per-request timeout (seconds)
	session=httpx.Client(), # custom httpx.Client (proxies, retries, etc.)
)
```

## Email Sending Examples

### Password reset template

```python
sm.send_email(
	toAddress="user@email.com",
	fromAddress="you@example.com",
	fromName="Your App",
	templateName="Password reset",
	templateProps={"resetUrl": "https://your.app/reset?token=123"},
)
```

### Schedule email delivery

```python
import datetime

scheduled_iso = (datetime.datetime.utcnow() + datetime.timedelta(minutes=60)).isoformat() + "Z"
sm.send_email(
	toAddress="user@email.com",
	fromAddress="your@startup.com",
	fromName="Startup name",
	templateName="Welcome",
	templateProps={"firstName": "Patrik"},
	scheduledAt=scheduled_iso,
)
```

### Template with dynamic list

```python
sm.send_email(
	toAddress="user@email.com",
	fromAddress="your@startup.com",
	fromName="Startup name",
	templateName="Template with dynamic list",
	templateProps={
		"list": [
			{"text": "Dynamic list"},
			{"text": "allows you to generate email template content"},
			{"text": "based on template props."},
		]
	},
)
```

### Custom HTML email

```python
sm.send_email(
	toAddress="user@email.com",
	fromAddress="your@startup.com",
	fromName="Startup name",
	subject="Testing html only custom emails :)",
	html="<html><body><h1>Hello world! 👋</h1></body></html>",
)
```

### Custom plain text email

```python
sm.send_email(
	toAddress="user@email.com",
	fromAddress="your@startup.com",
	fromName="Startup name",
	subject="Testing plain-text only custom emails :)",
	text="Hello world! 👋",
)
```

## Error handling

The SDK throws `SidemailError` for all errors. API errors include `message`, `httpStatus`, `errorCode`, and `moreInfo`.

```python
from sidemail import Sidemail, SidemailError

sm = Sidemail(api_key="your-api-key")
try:
	sm.send_email(
		toAddress="user@example.com",
		fromAddress="you@example.com",
		subject="Hello",
		text="Hello",
	)
except SidemailError as e:
	print(e.message)
	if e.httpStatus:  # API error
		print(e.httpStatus, e.errorCode, e.moreInfo)
```

## Response objects

Most responses are wrapped in a `Resource` enabling attribute access while remaining dict-like.

- Field names that collide with Python keywords or are invalid identifiers are suffixed with `_`.
- Methods return `Resource` wrappers (attribute + dict access); unwrap via `.to_dict()`.
- Original JSON available via `.raw`.

```python
email = sm.email.get("email-id")
print(email.id, email.status)
print(email["id"])          # dict-style
raw_json = email.raw         # original JSON mapping
flat_dict = email.to_dict()  # fully unwrapped dict
```

## Attachments helper

```python
from sidemail import Sidemail

with open("invoice.pdf", "rb") as f:
  attachment = Sidemail.file_to_attachment("invoice.pdf", f.read())

  sm.send_email(
    toAddress="user@email.com",
    fromAddress="you@example.com",
    subject="Invoice",
    text="Invoice attached.",
    attachments=[attachment],
  )
```

## Auto-pagination

List/search methods return a `QueryResult` containing the first page in `result.data`. Iterate across all pages with `auto_paginate()`.

```python
result = sm.contacts.list(limit=50)

for contact in result.auto_paginate():
	print(contact.emailAddress)
```

Callback-style iteration can be created easily with a helper (not built-in); use the for-loop above.

Supported auto-paging methods:

- `sm.contacts.list()`
- `sm.contacts.query()`
- `sm.email.search()`
- `sm.messenger.list()`

## Email Methods

### Search emails

Paginated (supports auto-pagination).

```python
result = sm.email.search(
	query={
		"toAddress": "john.doe@example.com",
		"status": "delivered",
		"templateProps": {"foo": "bar"},
	},
	limit=50,
)

print("First page count:", len(result.data))
for email in result.auto_paginate():
	print(email.id, email.status)
```

### Retrieve a specific email

```python
email = sm.email.get("SIDEMAIL_EMAIL_ID")
print("Email status:", email.status)
```

### Delete a scheduled email

Only scheduled (future) emails can be deleted.

```python
resp = sm.email.delete("SIDEMAIL_EMAIL_ID")
print("Deleted:", getattr(resp, "deleted", resp))
```

## Contact Methods

### Create or update a contact

```python
contact = sm.contacts.create_or_update(
	emailAddress="marry@lightning.com",
	identifier="123",
	customProps={
		"name": "Marry Lightning",
		# ... more props ...
	},
)
print("Contact status:", contact.status)
```

### Find a contact

```python
contact = sm.contacts.find("marry@lightning.com")
if contact:
	print("Found contact:", contact.emailAddress)
```

### List all contacts

```python
result = sm.contacts.list(limit=50)
print("Has more:", result.hasMore)
for c in result.auto_paginate():
	print(c.emailAddress)
```

### Query contacts (filtering)

```python
result = sm.contacts.query(limit=100, query={"customProps.plan": "pro"})
for c in result.auto_paginate():
	print(c.emailAddress)
```

### Delete a contact

```python
resp = sm.contacts.delete("marry@lightning.com")
print(resp)
```

## Project Methods

Linked projects are associated with the parent project of the API key used to initialize `Sidemail`. After creation, update the design to personalize templates.

### Create a linked project

```python
project = sm.project.create(name="Customer X linked project")
# Important! Save project.apiKey for later use
```

### Update a linked project

```python
updated = sm.project.update(
	name="New name",
	emailTemplateDesign={
		"logo": {
			"sizeWidth": 50,
			"href": "https://example.com",
			"file": "PHN2ZyBjbGlwLXJ1bGU9ImV2ZW5vZGQiIGZpbGwtcnVsZT0iZXZlbm9kZCIgc3Ryb2tlLWxpbmVqb2luPSJyb3VuZCIgc3Ryb2tlLW1pdGVybGltaXQ9IjIiIHZpZXdCb3g9IjAgMCAyNCAyNCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cGF0aCBkPSJtMTIgNS43MmMtMi42MjQtNC41MTctMTAtMy4xOTgtMTAgMi40NjEgMCAzLjcyNSA0LjM0NSA3LjcyNyA5LjMwMyAxMi41NC4xOTQuMTg5LjQ0Ni4yODMuNjk3LjI4M3MuNTAzLS4wOTQuNjk3LS4yODNjNC45NzctNC44MzEgOS4zMDMtOC44MTQgOS4zMDMtMTIuNTQgMC01LjY3OC03LjM5Ni02Ljk0NC0xMC0yLjQ2MXoiIGZpbGwtcnVsZT0ibm9uemVybyIvPjwvc3ZnPg==",
		},
		"font": {"name": "Acme"},
		"colors": {"highlight": "#0000FF", "isDarkModeEnabled": True},
		"unsubscribeText": "Darse de baja",
		"footerTextTransactional": "You're receiving these emails because you registered for Acme Inc.",
	},
)
```

### Get a project

```python
project = sm.project.get()
print(project.id, project.name)
```

### Delete a linked project

```python
resp = sm.project.delete()
print(resp)
```

### Messenger API (newsletters)

```python
result = sm.messenger.list(limit=20)
for m in result.auto_paginate():
	print(m.id, m.get("name"))

messenger = sm.messenger.get("messenger-id")
created = sm.messenger.create(subject="My Messenger", markdown="Broadcast message...")
updated = sm.messenger.update("messenger-id", name="Updated name")
deleted = sm.messenger.delete("messenger-id")
```

### Sending domains API

```python
domains = sm.domains.list()
domain = sm.domains.create(name="example.com")
deleted = sm.domains.delete("domain-id")
```

## More Info

Visit [Sidemail docs](https://sidemail.io/docs/) for more information.
