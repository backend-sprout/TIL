# Resolver VS Handler
## Resolver

* View resolvers: View resolvers are components capable of resolving view names to views
* Locale resolver: A locale resolver is a component capable of resolving the locale a client is using, in order to be able to offer internationalized views
* Theme resolver: A theme resolver is capable of resolving themes your web application can use, for example, to offer personalized layouts
* Multipart file resolver: A multipart file resolver offers the functionality to process file uploads from HTML forms
* Handler exception resolver(s): Handler exception resolvers offer functionality to map exceptions to views or implement other more complex exception handling code

## Handler

구성 가능한 핸들러 매핑,보기 해상도, 로케일 및 테마 해상도 및 파일 업로드 지원을 통해 핸들러에 요청을 발송하는 DispatcherServlet. 
기본 핸들러는 @Controller 및 @RequestMapping 주석을 기반으로하며 다양한 유연한 처리 방법을 제공합니다.
