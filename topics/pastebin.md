# Pastebin

## Requirements
- Upload or paste data and get a unique URL
- Only text
- Automatic data and link expiration (customizable)
- Custom alias


## Estimations
- Read-heavy servers
- Assume 5:1 ratio read-to-write

### Traffic:
- 1 mil new pastes every day -> 5 mil reads per day

New pastes/second:
`1M / (24 hrs * 3600 seconds) ~= 12 pastes/sec`

Paste reads/second:
`5M / (24 hrs * 3600 seconds) ~= 58 reads/sec`

### Storage
- Max of 10MB data
- Average contains 10 KB
- Total per day: `1M new pastes * 10 KB = 10 GB/day`
- Total over ten years: `36 TB`

1M pastes every day == 3.6B pastes in 10 years!

Generate and store keys to uniquely identify these pastes:
- Using base64 encoding ([A-Z, a-z, 0-9, ., -]), need 6 letter strings:
`64^6 ~= 68.7 billion unique strings`

- If 1 byte to store 1 char, total size to store 3.6B keys would be:
`3.6B * 6 = 22 GB`

- Keeping some margin, assume 70% capacity model, needs now at 51.4 TB

### Bandwidth
- Write requests: 12 new pastes per second * 10 KB each: `120 KB/s`
- Read requests: 58 reads/second * 10 KB each: `580 KB/s (0.6 MB/sec)`

### Memory
- Cache some of the hot pastes that are frequently accessed
- 20% of hot pastes generate 80% of traffic, cache 20%
- 5M read requests/day --> cache 20%:
`0.2 * 5M * 10KB ~= 10 GB`

## System APIs

```python
add_paste(
  api_dev_key, paste_data, custom_url=None, user_name=None, paste_name=None,
  expire_date=None
)
```
Returns -> successful insertion returns URL, otherwise error code

```py
get_paste(api_dev_key, api_paste_key)
```
- `api_paste_key` is a string representation of the paste, returns data

```py
delete_paste(api_dev_key, api_paste_key)
```
Returns -> `true` if successful, else `false`

## Database design

Some observations:
1. Billions of records
2. Each metadata object will be small (less than 100 bytes)
3. Each paste object is medium size (can be few MB)
4. No relationships between records, except user <-> Paste
5. Service is read heavy

### Schema
- Two tables, storing info about paste, and other is user's data

|     | Paste                    |
| --- | ---                      |
| PK  | **URLHash: varchar(16)** |
|     | ContentKey: varchar(512) |
|     | ExpirationDate: datetime |
|     | UserID: int              |
|     | CreationDate: Datetime   |

|     | User                   |
| --- | ---                    |
| PK  | **UserID: int**        |
|     | Name: varchar(20)      |
|     | Email: varchar(32)     |
|     | LastLogin: datetime    |
|     | CreationDate: datetime |

## High Level Design

1. Application layer (handles read and write requests)
2. Storage layer
    a. Database (metadata)
    b. Object storage (paste content)
