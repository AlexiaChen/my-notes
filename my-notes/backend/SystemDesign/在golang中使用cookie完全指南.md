
## 前言

在本篇文章中，我们将介绍如何在 Go 网络应用程序中使用 cookies，以便在特定客户端的 HTTP 请求之间持久保存数据。

在开始之前，推荐读一下MDN的[这一篇介绍]([Using HTTP cookies - HTTP | MDN (mozilla.org)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies))

文章涉及的全部代码都在[这里]([gist.github.com](https://gist.github.com/alexedwards/6d50c564dc597c243ce8b4aa2684d28f))


## 基本用法

首先要知道的是，Go 中的 Cookie 由 [http.Cookie 类型]([http package - net/http - Go Packages](https://pkg.go.dev/net/http#Cookie))表示。这是一个结构体，看起来像这样

```go
type Cookie struct {
    Name  string
    Value string

    Path       string    
    Domain     string    
    Expires    time.Time 
    RawExpires string   

    // MaxAge=0 means no 'Max-Age' attribute specified.
    // MaxAge<0 means delete cookie now, equivalently 'Max-Age: 0'
    // MaxAge>0 means Max-Age attribute present and given in seconds
    MaxAge   int 
    Secure   bool
    HttpOnly bool
    SameSite SameSite
    Raw      string
    Unparsed []string
}
```

-  `Name` 是cookie名字. 它包含任何 [US-ASCII characters](http://www.columbia.edu/kermit/ascii.html) 除了 `( ) < > @ , ; : \ " / [ ? ] = { }` 和空格, tab 还有 控制字符. 这是一个必填字段
- `Value` 包含要持久化的数据。它可以包含任何 US-ASCII 字符，但` , ; \ " ` 以及空格、制表符和控制字符除外。这是一个必填字段。
- `Path`、`Domain`、`Expires`、`MaxAge`、`Secure`、`HttpOnly` 和 `SameSite` 可直接映射到相应的 [cookie 属性]([HTTP cookie - Wikipedia](https://en.wikipedia.org/wiki/HTTP_cookie#Cookie_attributes))。所有这些都是可选字段。
- 如果设置`SameSite` 字段的值应为 `net/http` 软件包中的 SameSite 常量之一。
- `RawExpires`、`Raw` 和 `Unparsed` 字段仅在 Go 程序作为客户端（而非服务器）从 HTTP 响应中解析 cookie 时使用。大多数情况下，你不需要使用这些字段。

Cookie 可以使用 `http.SetCookie()`  [http package - net/http - Go Packages](https://pkg.go.dev/net/http#SetCookie) 函数写入 HTTP 响应，也可以使用 `*Request.Cookie()`  方法从 HTTP 请求中读取。

让我们在一个工作示例中使用这些功能。

如果你想跟着学，请看以下go项目

```go
package main

import (
    "errors"
    "log"
    "net/http"
)

func main() {
    // Start a web server with the two endpoints.
    mux := http.NewServeMux()
    mux.HandleFunc("/set", setCookieHandler)
    mux.HandleFunc("/get", getCookieHandler)

    log.Print("Listening...")
    err := http.ListenAndServe(":3000", mux)
    if err != nil {
        log.Fatal(err)
    }
}

func setCookieHandler(w http.ResponseWriter, r *http.Request) {
    // Initialize a new cookie containing the string "Hello world!" and some
    // non-default attributes.
    cookie := http.Cookie{
        Name:     "exampleCookie",
        Value:    "Hello world!",
        Path:     "/",
        MaxAge:   3600,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteLaxMode,
    }

    // Use the http.SetCookie() function to send the cookie to the client.
    // Behind the scenes this adds a `Set-Cookie` header to the response
    // containing the necessary cookie data.
    http.SetCookie(w, &cookie)

    // Write a HTTP response as normal.
    w.Write([]byte("cookie set!"))
}

func getCookieHandler(w http.ResponseWriter, r *http.Request) {
    // Retrieve the cookie from the request using its name (which in our case is
    // "exampleCookie"). If no matching cookie is found, this will return a
    // http.ErrNoCookie error. We check for this, and return a 400 Bad Request
    // response to the client.
    cookie, err := r.Cookie("exampleCookie")
    if err != nil {
        switch {
        case errors.Is(err, http.ErrNoCookie):
            http.Error(w, "cookie not found", http.StatusBadRequest)
        default:
            log.Println(err)
            http.Error(w, "server error", http.StatusInternalServerError)
        }
        return
    }

    // Echo out the cookie value in the response body.
    w.Write([]byte(cookie.Value))
}
```

> 另外，我们将 cookie 的安全属性设置为 true。这向客户端（通常是网络浏览器）表明，cookie 只能用于 "安全 "连接。一般来说，这意味着 cookie 只能用于加密连接（即 HTTPS），但许多现代浏览器（包括 Firefox 和 Chrome 浏览器）也将与本地主机的未加密连接视为 "安全 "连接。这意味着，即使我们的网络应用程序只使用 HTTP，cookie 也应在 localhost 上运行。

运行这个go程序，然后在浏览器中打开 http://localhost:3000/set 你应该能看到 "cookie set！"的响应，如果你打开了开发工具，还应该能看到包含 HTTP 响应头中数据的 `Set-Cookie` 头。

![[Pasted image 20231110141415.png]]


然后，如果您访问 http://localhost:3000/get 我们的 `exampleCookie` cookie 将与 HTTP 请求一起传回，我们的 `getCookieHandler` 将检索 cookie 值并将其打印在响应中。就像这样

![[Pasted image 20231110141640.png]]

如果需要，也可以使用 curl 向 http://localhost:3000/set 提出请求，查看 `Set-Cookie` 标头的内容。就像这样

```bash
curl -i http://localhost:3000/set
HTTP/1.1 200 OK
Set-Cookie: exampleCookie="Hello world!"; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
Date: Sun, 25 Sep 2022 08:45:02 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

cookie set!
```


## 编码特殊字符和最大长度

到目前为止，一切顺利！但在编写 cookie 时，有几件重要的事情需要注意。

正如上文简要提到的，cookie 值必须只包含 US-ASCII 字符的子集。如果尝试使用不支持的字符，Go 会在设置 `Set-Cookie` 头之前将其删除。

让我们尝试一下，调整 `setCookieHandler`，写入一个包含非 US-ASCII 字符的 cookie 值，如 ``"Hello Zoë!"``（请注意 "Hello Zoë! (注意其中的 `ë` 字符）：

```go
package main

...

func setCookieHandler(w http.ResponseWriter, r *http.Request) {
    cookie := http.Cookie{
        Name:     "exampleCookie",
        Value:    "Hello Zoë!",
        Path:     "/",
        MaxAge:   3600,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteLaxMode,
    }

    http.SetCookie(w, &cookie)

    w.Write([]byte("cookie set!"))
}
```

然后，当您向 http://localhost:3000/set 提出请求时，您会看到 cookie 值被精简为 `"Hello Zo!"`。

```bash
 curl -i http://localhost:3000/set
HTTP/1.1 200 OK
Set-Cookie: exampleCookie="Hello Zo!"; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
Date: Sun, 25 Sep 2022 09:00:03 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

cookie set!
```

避免此类问题的好办法是在写入 cookie 值之前对其进行 base64 编码。因为 base64 字符集是 Cookie 中支持的 US-ASCII 字符的子集，所以我们可以确信 Cookie 值中不会有任何内容被剥离。

另一个需要注意的问题是，网络浏览器会对 Cookie 设置最大大小限制。但这一限制以及 cookie 大小的计算方法取决于所使用的浏览器版本。为防止出现问题，一个好的经验法则是将 cookie 的总大小（包括所有属性）控制在 4096 字节以内。

如果尝试发送大于 4096 字节的 cookie，Go 会顺利写入 `Set-Cookie` 头（不会被截断），但客户端有可能截断或拒绝 cookie。

为了解决这两个潜在问题，让我们创建一个包含几个辅助函数的`internal/cookies` 包：

- Write() 函数将 cookie 值编码为 base64，并在写入之前检查 cookie 的总长度是否超过 4096 字节。
- Read() 函数用于从当前请求中读取 cookie，并将 cookie 值解码为 base64。

```go
package cookies

import (
    "encoding/base64"
    "errors"
    "net/http"
)

var (
    ErrValueTooLong = errors.New("cookie value too long")
    ErrInvalidValue = errors.New("invalid cookie value")
)

func Write(w http.ResponseWriter, cookie http.Cookie) error {
    // Encode the cookie value using base64.
    cookie.Value = base64.URLEncoding.EncodeToString([]byte(cookie.Value))

    // Check the total length of the cookie contents. Return the ErrValueTooLong
    // error if it's more than 4096 bytes.
    if len(cookie.String()) > 4096 {
        return ErrValueTooLong
    }

    // Write the cookie as normal.
    http.SetCookie(w, &cookie)

    return nil
}

func Read(r *http.Request, name string) (string, error) {
    // Read the cookie as normal.
    cookie, err := r.Cookie(name)
    if err != nil {
        return "", err
    }

    // Decode the base64-encoded cookie value. If the cookie didn't contain a
    // valid base64-encoded value, this operation will fail and we return an
    // ErrInvalidValue error.
    value, err := base64.URLEncoding.DecodeString(cookie.Value)
    if err != nil {
        return "", ErrInvalidValue
    }

    // Return the decoded cookie value.
    return string(value), nil
}
```

main.go的内容:

```go
package main

import (
    "errors"
    "log"
    "net/http"

    "example.com/example-project/internal/cookies" // Import the internal/cookies package.
)

...

func setCookieHandler(w http.ResponseWriter, r *http.Request) {
    // Initialize the cookie as normal.
    cookie := http.Cookie{
        Name:     "exampleCookie",
        Value:    "Hello Zoë!",
        Path:     "/",
        MaxAge:   3600,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteLaxMode,
    }

    // Write the cookie. If there is an error (due to an encoding failure or it
    // being too long) then log the error and send a 500 Internal Server Error
    // response.
    err := cookies.Write(w, cookie)
    if err != nil {
        log.Println(err)
        http.Error(w, "server error", http.StatusInternalServerError)
        return
    }

    w.Write([]byte("cookie set!"))
}

func getCookieHandler(w http.ResponseWriter, r *http.Request) {
    // Use the Read() function to retrieve the cookie value, additionally
    // checking for the ErrInvalidValue error and handling it as necessary.
    value, err := cookies.Read(r, "exampleCookie")
    if err != nil {
        switch {
        case errors.Is(err, http.ErrNoCookie):
            http.Error(w, "cookie not found", http.StatusBadRequest)
        case errors.Is(err, cookies.ErrInvalidValue):
            http.Error(w, "invalid cookie", http.StatusBadRequest)
        default:
            log.Println(err)
            http.Error(w, "server error", http.StatusInternalServerError)
        }
        return
    }

    w.Write([]byte(value))
}
```

如果您重新启动Web应用程序并在浏览器中发出对 http://localhost:3000/set 的请求，然后紧接着发出对 http://localhost:3000/get 的请求，您现在应该能够成功地完整地看到消息`“Hello Zoë！”`。

![[Pasted image 20231110142830.png]]

同样，如果使用 curl 向 http://localhost:3000/set 提出请求，就会看到 cookie 值为 `SGVsbG8gWm_DqyE=` - 这是 `Hello Zoë` 的 base64 编码。

```bash
curl -i localhost:3000/set
HTTP/1.1 200 OK
Set-Cookie: exampleCookie=SGVsbG8gWm_DqyE=; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
Date: Sun, 25 Sep 2022 09:14:18 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

cookie set
```


```bash
echo "SGVsbG8gWm_DqyE=" | base64url --decode
Hello Zoë!
```


## 防伪(签名)Cookies

默认情况下，你不应该信任 cookie 数据。因为 Cookie 存储在客户端，用户很容易对其进行编辑（事实上，许多网络浏览器扩展正是为此目的而存在的）。因此，如果您要根据 cookie 的值在网络应用程序中执行操作，首先必须验证 cookie 是否被编辑过，并包含您设置的原始名称和值。

一个好的方法是生成 cookie 名称和值的 [HMAC 签名]([What is HMAC(Hash based Message Authentication Code)? - GeeksforGeeks](https://www.geeksforgeeks.org/what-is-hmachash-based-message-authentication-code/))，然后在将 cookie 值发送给客户端之前，将此签名预置到 cookie 值中。这样，最终值就会是这种格式：

`cookie.Value = "{HMAC signature}{original value}"`

当我们收到客户端发回的 cookie 时，我们可以根据 cookie 名称和原始值重新计算 HMAC 签名，并检查重新计算的 HMAC 签名是否与收到的 cookie 开头的签名相匹配。如果它们匹配，就确认了名称和值的完整性，我们也就知道它没有被客户端编辑过。

让我们更新`inertnal/cookies/cookies.go` 文件，在其中加入一些 `WriteSigned()` 和 `ReadSigned()` 函数，它们的作用正是如此。

```go
package cookies

import (
    "crypto/hmac"  
    "crypto/sha256"
    "encoding/base64"
    "errors"
    "net/http"
)

...

func WriteSigned(w http.ResponseWriter, cookie http.Cookie, secretKey []byte) error {
    // Calculate a HMAC signature of the cookie name and value, using SHA256 and
    // a secret key (which we will create in a moment).
    mac := hmac.New(sha256.New, secretKey)
    mac.Write([]byte(cookie.Name))
    mac.Write([]byte(cookie.Value))
    signature := mac.Sum(nil)

    // Prepend the cookie value with the HMAC signature.
    cookie.Value = string(signature) + cookie.Value

    // Call our Write() helper to base64-encode the new cookie value and write
    // the cookie.
    return Write(w, cookie)
}

func ReadSigned(r *http.Request, name string, secretKey []byte) (string, error) {
    // Read in the signed value from the cookie. This should be in the format
    // "{signature}{original value}".
    signedValue, err := Read(r, name)
    if err != nil {
        return "", err
    }

    // A SHA256 HMAC signature has a fixed length of 32 bytes. To avoid a potential
    // 'index out of range' panic in the next step, we need to check sure that the
    // length of the signed cookie value is at least this long. We'll use the 
    // sha256.Size constant here, rather than 32, just because it makes our code
    // a bit more understandable at a glance.
    if len(signedValue) < sha256.Size {
        return "", ErrInvalidValue
    }

    // Split apart the signature and original cookie value.
    signature := signedValue[:sha256.Size]
    value := signedValue[sha256.Size:]

    // Recalculate the HMAC signature of the cookie name and original value.
    mac := hmac.New(sha256.New, secretKey)
    mac.Write([]byte(name))
    mac.Write([]byte(value))
    expectedSignature := mac.Sum(nil)

    // Check that the recalculated signature matches the signature we received
    // in the cookie. If they match, we can be confident that the cookie name
    // and value haven't been edited by the client.
    if !hmac.Equal([]byte(signature), expectedSignature) {
        return "", ErrInvalidValue
    }

    // Return the original cookie value.
    return value, nil
}


```

好了，让我们更新 main.go 文件，加入秘钥并使用新的 `WriteSigned()` 和 `ReadSigned()` 函数。
秘钥应使用加密安全随机数生成器（CSRNG）生成，对你的应用程序来说应该是唯一的，最好至少有 32 字节的熵。在本例中，我们将使用一个随机的 64 个字符的十六进制字符串，并对其进行解码，从而得到一个包含 32 个随机字节的字节片。

```go
package main

import (
    "encoding/hex"
    "errors"
    "log"
    "net/http"

    "example.com/example-project/internal/cookies"
)

// Declare a global variable to hold the secret key.
var secretKey []byte

func main() {
    var err error

    // Decode the random 64-character hex string to give us a slice containing
    // 32 random bytes. For simplicity, I've hardcoded this hex string but in a
    // real application you should read it in at runtime from a command-line
    // flag or environment variable.
    secretKey, err = hex.DecodeString("13d6b4dff8f84a10851021ec8608f814570d562c92fe6b5ec4c9f595bcb3234b")
    if err != nil {
        log.Fatal(err)
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/set", setCookieHandler)
    mux.HandleFunc("/get", getCookieHandler)

    log.Print("Listening...")
    err = http.ListenAndServe(":3000", mux)
    if err != nil {
        log.Fatal(err)
    }
}

func setCookieHandler(w http.ResponseWriter, r *http.Request) {
    cookie := http.Cookie{
        Name:     "exampleCookie",
        Value:    "Hello Zoë!",
        Path:     "/",
        MaxAge:   3600,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteLaxMode,
    }

    // Use the WriteSigned() function, passing in the secret key as the final
    // argument.
    err := cookies.WriteSigned(w, cookie, secretKey)
    if err != nil {
        log.Println(err)
        http.Error(w, "server error", http.StatusInternalServerError)
        return
    }

    w.Write([]byte("cookie set!"))
}

func getCookieHandler(w http.ResponseWriter, r *http.Request) {
    // Use the ReadSigned() function, passing in the secret key as the final
    // argument.
    value, err := cookies.ReadSigned(r, "exampleCookie", secretKey)
    if err != nil {
        switch {
        case errors.Is(err, http.ErrNoCookie):
            http.Error(w, "cookie not found", http.StatusBadRequest)
        case errors.Is(err, cookies.ErrInvalidValue):
            http.Error(w, "invalid cookie", http.StatusBadRequest)
        default:
            log.Println(err)
            http.Error(w, "server error", http.StatusInternalServerError)
        }
        return
    }

    w.Write([]byte(value))
}
```

如果您在浏览器中访问 http://localhost:3000/set ，然后再访问 http://localhost:3000/get ，您应该仍能成功看到 "Hello Zoë!"的信息。

如果您愿意，也可以使用浏览器扩展来更改 cookie 值（在浏览器扩展商店中搜索 "cookie 编辑器"）。这样做之后，再次访问 http://localhost:3000/get 时，就会收到 400 Bad Request 响应和 "invalid cookie "信息。

在继续之前，让我们使用 curl 向 http://localhost:3000/set 提出请求：

```bash
 curl -i http://localhost:3000/set
HTTP/1.1 200 OK
Set-Cookie: exampleCookie=1lYrR9MfMsu6Dm39EgfbOuFTUbZm3_5tmWsF943HN4hIZWxsbyBab8OrIQ==; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
Date: Wed, 28 Sep 2022 09:28:55 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

cookie set!
```


在我的案例中，我们可以看到已签名的 cookie 值是

`1lYrR9MfMsu6Dm39EgfbOuFTUbZm3_5tmWsF943HN4hIZWxsbyBab8OrIQ==`

```bash
echo "1lYrR9MfMsu6Dm39EgfbOuFTUbZm3_5tmWsF943HN4hIZWxsbyBab8OrIQ==" | base64url --decode
�V+G�2˺m��:�SQ�f��m�k���7�Hello Zoë!
```

解码值的第一部分是 HMAC 签名（看起来像胡言乱语），然后是明文的原始 cookie 值。

## 保密(加密)和仿篡改的cookies

上述 HMAC 签名模式非常适合以下情况：当你想确认 cookie 没有被客户端编辑过，而且你不担心客户端能读取 cookie 数据（即 cookie 不包含任何秘密或机密信息）。

但如果你确实想防止客户端读取 cookie 数据，我们就需要在写入数据前对其进行加密。

对 Cookie 中的数据进行加密的一个好方法是使用 [AES-GCM]([AES-GCM authenticated encryption (cryptosys.net)](https://www.cryptosys.net/pki/manpki/pki_aesgcmauthencryption.html))（具有伽罗瓦/计数器模式的 AES）加密。AES-GCM 是一种经过验证的加密方式，它既能加密数据，又能验证数据，因此效果很好。加密可确保数据的保密性，而验证可确保数据的完整性（即数据未被更改）。

实际上，使用 AES-GCM 对 cookie 数据进行加密是一种相对简单的方法，只需一个步骤就能为我们提供保密、防篡改的 cookie。

让我们创建两个新的辅助函数 `WriteEncrypted()` 和 `ReadEncrypted()` 来使用它。就像这样

```go
package cookies

import (
    "crypto/aes"    
    "crypto/cipher" 
    "crypto/hmac"
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "errors"
    "fmt" 
    "io" 
    "net/http"
    "strings" 
)

...

func WriteEncrypted(w http.ResponseWriter, cookie http.Cookie, secretKey []byte) error {
    // Create a new AES cipher block from the secret key.
    block, err := aes.NewCipher(secretKey)
    if err != nil {
        return err
    }

    // Wrap the cipher block in Galois Counter Mode.
    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return err
    }

    // Create a unique nonce containing 12 random bytes.
    nonce := make([]byte, aesGCM.NonceSize())
    _, err = io.ReadFull(rand.Reader, nonce)
    if err != nil {
        return err
    }

    // Prepare the plaintext input for encryption. Because we want to
    // authenticate the cookie name as well as the value, we make this plaintext
    // in the format "{cookie name}:{cookie value}". We use the : character as a
    // separator because it is an invalid character for cookie names and
    // therefore shouldn't appear in them.
    plaintext := fmt.Sprintf("%s:%s", cookie.Name, cookie.Value)

    // Encrypt the data using aesGCM.Seal(). By passing the nonce as the first
    // parameter, the encrypted data will be appended to the nonce — meaning
    // that the returned encryptedValue variable will be in the format
    // "{nonce}{encrypted plaintext data}".
    encryptedValue := aesGCM.Seal(nonce, nonce, []byte(plaintext), nil)

    // Set the cookie value to the encryptedValue.
    cookie.Value = string(encryptedValue)

    // Write the cookie as normal.
    return Write(w, cookie)
}

func ReadEncrypted(r *http.Request, name string, secretKey []byte) (string, error) {
    // Read the encrypted value from the cookie as normal.
    encryptedValue, err := Read(r, name)
    if err != nil {
        return "", err
    }

    // Create a new AES cipher block from the secret key.
    block, err := aes.NewCipher(secretKey)
    if err != nil {
        return "", err
    }

    // Wrap the cipher block in Galois Counter Mode.
    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    // Get the nonce size.
    nonceSize := aesGCM.NonceSize()

    // To avoid a potential 'index out of range' panic in the next step, we
    // check that the length of the encrypted value is at least the nonce
    // size.
    if len(encryptedValue) < nonceSize {
        return "", ErrInvalidValue
    }

    // Split apart the nonce from the actual encrypted data.
    nonce := encryptedValue[:nonceSize]
    ciphertext := encryptedValue[nonceSize:]

    // Use aesGCM.Open() to decrypt and authenticate the data. If this fails,
    // return a ErrInvalidValue error.
    plaintext, err := aesGCM.Open(nil, []byte(nonce), []byte(ciphertext), nil)
    if err != nil {
        return "", ErrInvalidValue
    }

    // The plaintext value is in the format "{cookie name}:{cookie value}". We
    // use strings.Cut() to split it on the first ":" character.
    expectedName, value, ok := strings.Cut(string(plaintext), ":")
    if !ok {
        return "", ErrInvalidValue
    }

    // Check that the cookie name is the expected one and hasn't been changed.
    if expectedName != name {
        return "", ErrInvalidValue
    }

    // Return the plaintext cookie value.
    return value, nil
}
```

对应的main.go是:

```go
package main

...

func setCookieHandler(w http.ResponseWriter, r *http.Request) {
    cookie := http.Cookie{
        Name:     "exampleCookie",
        Value:    "Hello Zoë!",
        Path:     "/",
        MaxAge:   3600,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteLaxMode,
    }

    err := cookies.WriteEncrypted(w, cookie, secretKey)
    if err != nil {
        log.Println(err)
        http.Error(w, "server error", http.StatusInternalServerError)
        return
    }

    w.Write([]byte("cookie set!"))
}

func getCookieHandler(w http.ResponseWriter, r *http.Request) {
    value, err := cookies.ReadEncrypted(r, "exampleCookie", secretKey)
    if err != nil {
        switch {
        case errors.Is(err, http.ErrNoCookie):
            http.Error(w, "cookie not found", http.StatusBadRequest)
        case errors.Is(err, cookies.ErrInvalidValue):
            http.Error(w, "invalid cookie", http.StatusBadRequest)
        default:
            log.Println(err)
            http.Error(w, "server error", http.StatusInternalServerError)
        }
        return
    }

    w.Write([]byte(value))
}
```

> 使用 AES-GCM 加密时，密钥长度必须正好为 32 字节。否则会出现 `crypto/aes: invalid key size \<N\>` 这样的运行时错误。

同样，你可以在浏览器中访问 http://localhost:3000/set 和 http://localhost:3000/get 应该还是能成功看到 "Hello Zoë!"的信息。如果你使用浏览器扩展编辑 `exampleCookie` cookie，你会发现任何后续请求都会得到 "invalid cookie "的响应。

现在让我们用 curl 来看看 `Set-Cookie` 标头。

```bash
curl -i http://localhost:3000/set
HTTP/1.1 200 OK
Set-Cookie: exampleCookie=hBGecbVJ2cI0yAwrbMYd5sv7qslxBJoGnk7LBLHVR9rKrqh1cTVs2IuWHZUOkl2fdYeIYmY=; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax
Date: Wed, 28 Sep 2022 10:37:23 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

cookie set!
```

加密的cookie的base64编码是:

`hBGecbVJ2cI0yAwrbMYd5sv7qslxBJoGnk7LBLHVR9rKrqh1cTVs2IuWHZUOkl2fdYeIYmY=`

解码是:

```bash
 echo "hBGecbVJ2cI0yAwrbMYd5sv7qslxBJoGnk7LBLHVR9rKrqh1cTVs2IuWHZUOkl2fdYeIYmY=" | base64url --decode
��q�I��4�+l������q��N���G�ʮ�uq5l؋���]�u��bf
```

