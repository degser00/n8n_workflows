## OpenArchiver API – Retrieve Emails and Attachments by Date

This document describes how to retrieve archived emails and their attachments from OpenArchiver **using only the public API**.

Attachments are **stored separately** from email bodies and must be downloaded explicitly.

---

## 1. API Configuration

- **Base URL:** `http://<your-instance-url>/api/v1`
- **Authentication Header:**
  ```
  X-API-KEY: <your_generated_key>
  ```

---

## 2. Search Emails by Date

Use the Search API to retrieve email metadata for a specific date or date range.

- **Endpoint**
  ```
  GET /api/v1/search
  ```

- **Query Parameters**
  - `keywords`: `*` (match all emails)
  - `filters`: date filter
  - `limit`: page size (pagination required)

### Example: Search Emails for a Specific Day

```bash
curl -X GET "http://localhost:3000/api/v1/search?keywords=*&filters=date='2024-05-20'&limit=1000" \
  -H "X-API-KEY: your_api_key_here"
```

This call returns **metadata only**, including the email `id`.

---

## 3. Retrieve Full Email Metadata

For each email ID returned by the search:

```
GET /api/v1/archived-emails/{emailId}
```

This response includes:
- Email metadata
- Raw email content
- Attachment list with `fileId` references

---

## 4. Download Attachments

Attachments are not embedded in `.eml` files by default.

For each attachment `fileId`:

```
GET /api/v1/storage/download/{fileId}
```

This endpoint returns the attachment binary.

---

## Operational Notes

- Attachments must always be handled explicitly
- Pagination is required for large archives
- Date field name (`date`) must match the OpenArchiver search index
- API-based export avoids filesystem and database coupling

---

## References

- https://docs.openarchiver.com/api/search.html
