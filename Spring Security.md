# Spring Security

## 安全模块工作流程

1. 访问接口时，会被Spring Security拦截认证，如果发现用户没有携带有效的cookie认证授权，就会自动重定向到Spring Security的默认登陆界面http://127.0.0.1:8080/login
2. 如果用户有携带有效的cookie认证授权，Spring Security会重新设置cookie，并重定向到访问的接口地址

