
当用Flask或者Django的时候，内建的WSGI server是不推荐部署在生产环境的，只是开发调试用。因为性能的各种原因，所以需要用Gunicorn和uWSGI这样的生产级的WSGI server代替。

所以下面推荐两篇Gunicorn的配置文章，和相应的配置应用场景

[Better performance by optimizing Gunicorn config | by Omar Rayward | Building the system | Medium](https://medium.com/building-the-system/gunicorn-3-means-of-concurrency-efbb547674b7)

[How to choose the right Gunicorn Worker Type | Medium](https://luis-sena.medium.com/gunicorn-worker-types-youre-probably-using-them-wrong-381239e13594)

