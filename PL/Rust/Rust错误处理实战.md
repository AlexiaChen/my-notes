简单来说就是library用thiserror这个crate， binary工程用anyhow这个crate。

library 一般是给外部调用，所以需要尽可能enum一个对外易懂的错误，因为library的错误是API的一部分，caller需要知道该怎么处理错误，是错误的消费者。所以对于library，尽可能不用这样的返回Boxed错误 `Result<String, Box<dyn std::error::Error>> `,  目前就是用thiserror了，自己enum出来一些你自定义的Error返回出去，让caller方便处理。至于binary的话，因为再上一层就是终端用户了，用Boxed Error也是可以的，大部分错误都可以不用处理，而是打印日志或者弹出错误框什么的。

## References

- https://nick.groenen.me/posts/rust-error-handling/
- https://www.sheshbabu.com/posts/rust-error-handling/
