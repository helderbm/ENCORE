# JSON API REST schemes

The beginning of an awesome article...

## `versions` response schema

```json json_schema
{
 "type": "object",
        "description": "JSON object for the API vertions",
        "properties": {
          "version": {
            "type": "string",
            "description": "vaersion number"
          },
          "date": {
            "type": "string",
            "description": "version date"
          },
          "changes": {
            "type": "string",
            "description": "changes since last version"
          }
        },
        "required": [
          "version",
          "date",
          "changes"
        ]        
}
```

```json http
{
  "method": "get",
  "url": "https://spacloud.eu/public/9786582aaf93e/versions",
  "headers": {
    "token": "234dsfv443",
    "request-id": "e3425gfedgrf"
  }
}
```
