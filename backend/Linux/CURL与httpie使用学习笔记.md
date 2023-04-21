
## CURL

```bash
curl -X 'POST' \
  'https://api.zealy.io/communities/gabbyworld/users/c079e0eb-7a33-4b2a-9b7c-3e2f643e6136/xp' \
  -H 'accept: application/json' \
  -H 'x-api-key: d13e57hvy8DaWijFSSfTt5nxsnE' \
  -H 'Content-Type: application/json' \
  -d '{
  "xp": 1,
  "label": "Test"
}'

curl -X 'POST' \
  'http://127.0.0.1:5000/api/chat' \
  -H 'accept: application/json' \
  -H 'X-Api-Key-Id: LocalGabby' \
  -H 'X-Api-Key: LocalGabbyAPIKey' \
  -H 'Content-Type: application/json' \
  -d '{
  "userid": "mygabbykk",
  "context": [
    "Who is Elon Musk?"
  ]
}'


curl -X 'DELETE' \
  'https://api.zealy.io/communities/gabbyworld/users/c079e0eb-7a33-4b2a-9b7c-3e2f643e6136/xp' \
  -H 'accept: application/json' \
  -H 'x-api-key: d13e57hvy8DaWijFSSfTt5nxsnE' \
  -H 'Content-Type: application/json' \
  -d '{
  "xp": 1,
  "label": "Test"
}'

curl -X 'GET' \
  'https://api.zealy.io/communities/gabbyworld/users?ethAddress=0xe369189543E8C4a27a8938168e1Be303934f16f6' \
  -H 'accept: application/json' \
  -H 'x-api-key: d13e57hvy8DaWijFSSfTt5nxsnE' 

```

## Httpie

因为curl不能格式化返回的json串，使用不便，而且不是很简洁，我很喜欢httpie

```bash
sudo apt install httpie
```

```bash
# headers可以直接以Key:Value的形式设置，默认是GET。GET的请求参数是param==value 这样的形式
http https://api.zealy.io/communities/gabbyworld/leaderboard x-api-key:d13e57hvy8DaWijFSSfTt5nxsnE page==1 limit==10

# 表单
http -f POST pie.dev/post hello=World
http -f POST pie.dev/post name='John Smith' cv@~/files/data.xml

# POST JSON
http POST pie.dev/post X-API-Token:123 name=John



```

```bash
http PUT pie.dev/put \
    name=John \                        # String (default)
    age:=29 \                          # Raw JSON — Number
    married:=false \                   # Raw JSON — Boolean
    hobbies:='["http", "pies"]' \      # Raw JSON — Array
    favorite:='{"tool": "HTTPie"}' \   # Raw JSON — Object
    bookmarks:=@files/data.json \      # Embed JSON file
    description=@files/text.txt        # Embed text file
```

```json
{
    "age": 29,
    "hobbies": [
        "http",
        "pies"
    ],
    "description": "John is a nice guy who likes pies.",
    "married": false,
    "name": "John",
    "favorite": {
        "tool": "HTTPie"
    },
    "bookmarks": {
        "HTTPie": "https://httpie.org",
    }
}
```

```bash
http pie.dev/post \
  platform[name]=HTTPie \
  platform[about][mission]='Make APIs simple and intuitive' \
  platform[about][homepage]=httpie.io \
  platform[about][homepage]=httpie.io \
  platform[about][stars]:=54000 \
  platform[apps][]=Terminal \
  platform[apps][]=Desktop \
  platform[apps][]=Web \
  platform[apps][]=Mobile
```

```json
{
    "platform": {
        "name": "HTTPie",
        "about": {
            "mission": "Make APIs simple and intuitive",
            "homepage": "httpie.io",
            "stars": 54000
        },
        "apps": [
            "Terminal",
            "Desktop",
            "Web",
            "Mobile"
        ]
    }
}
```
