# Flask 源码分析(一) - app调用流程

Flask.run -> werkzeug.run_simple -> werkzeug.make_server -> serve_forever

过程中涉及的类关系如下:

- werkzeug.BaseWSGIServer < http.HTTPServer < socketserver.TCPServer < socketserver.BaseServer
- werkzeug.WSGIRequestHandler < http.BaseHTTPRequestHandler < socketserver.StreamRequestHandler < socketserver.BaseRequestHandler

![classes_socket-http-wsgi](images/classes_socket-http-wsgi.svg "")

<div style='display: none'>
```plantuml
@startuml socket-http-wsgi
'flask
socketserver.BaseServer <|-- socketserver.TCPServer
socketserver.TCPServer <|-- http.HTTPServer
http.HTTPServer <|-- werkzeug.BaseWSGIServer

socketserver.BaseRequestHandler <|-- socketserver.StreamRequestHandler
socketserver.StreamRequestHandler <|-- http.BaseHTTPRequestHandler
http.BaseHTTPRequestHandler <|-- werkzeug.WSGIRequestHandler

class socketserver.BaseServer {
    RequestHandlerClass
    server_address
    timeout : NoneType
    close_request()
    finish_request()
    handle_error()
    handle_request()
    handle_timeout()
    process_request()
    serve_forever()
    server_activate()
    server_close()
    service_actions()
    shutdown()
    shutdown_request()
    verify_request()
}
class socketserver.TCPServer {
    address_family : int
    allow_reuse_address : bool
    request_queue_size : int
    server_address
    socket : socket
    socket_type : int
    close_request()
    fileno()
    get_request()
    server_activate()
    server_bind()
    server_close()
    shutdown_request()
}
class socketserver.BaseRequestHandler {
    client_address
    request
    server
    finish()
    handle()
    setup()
}
class socketserver.StreamRequestHandler {
    connection
    disable_nagle_algorithm : bool
    rbufsize : int
    rfile
    timeout : NoneType
    wbufsize : int
    wfile
    finish()
    setup()
}
class http.HTTPServer {
    allow_reuse_address : int
    server_name
    server_port
    server_bind()
}
class http.BaseHTTPRequestHandler {
    MessageClass : HTTPMessage
    close_connection : bool
    command : str, NoneType
    default_request_version : str
    error_content_type : str
    error_message_format : str
    headers
    monthname : list
    path
    protocol_version : str
    raw_requestline
    request_version : str
    requestline : str
    responses
    server_version : str
    sys_version
    weekdayname : list
    address_string()
    date_time_string()
    end_headers()
    flush_headers()
    handle()
    handle_expect_100()
    handle_one_request()
    log_date_time_string()
    log_error()
    log_message()
    log_request()
    parse_request()
    send_error()
    send_header()
    send_response()
    send_response_only()
    version_string()
}
class werkzeug.BaseWSGIServer {
    address_family : int
    app
    host
    multiprocess : bool
    multithread : bool
    passthrough_errors : bool
    port
    request_queue_size : int
    server_address
    shutdown_signal : bool
    socket : socket
    ssl_context : NoneType
    get_request()
    handle_error()
    log()
    serve_forever()
}
class werkzeug.WSGIRequestHandler {
    client_address : tuple, str
    close_connection : bool, int
    environ : dict
    raw_requestline
    server_version
    address_string()
    connection_dropped()
    get_header_items()
    handle()
    handle_one_request()
    initiate_shutdown()
    log()
    log_error()
    log_message()
    log_request()
    make_environ()
    port_integer()
    run_wsgi()
    send_response()
    version_string()
}
@enduml
```
</div>

详细调用序列图

![flask-callflow](images/flask-callflow.svg "")

<div style='display: none'>
```plantuml
@startuml flask-callflow
autonumber
main -> FlaskApp: run
    FlaskApp -> werkzeug: run_simple
    note left: application:self
        werkzeug -> werkzeug: make_server
            werkzeug -> werkzeug.BaseWSGIServer: __init__
            note right: handler:WSGIRequestHandler
                werkzeug.BaseWSGIServer -> http.HTTPServer: __init__
                http.HTTPServer --> socketserver.TCPServer: __init__
                note right: RequestHandlerClass:WSGIRequestHandler
                    socketserver.TCPServer -> socketserver.BaseServer: __init__
                    socketserver.TCPServer -> http.HTTPServer: server_bind
                    'note left: override
                    socketserver.TCPServer -> socketserver.TCPServer: server_activate
        werkzeug -> werkzeug.BaseWSGIServer: serve_forever
            werkzeug.BaseWSGIServer -> http.HTTPServer: serve_forever
            http.HTTPServer --> socketserver.TCPServer: serve_forever
            socketserver.TCPServer --> socketserver.BaseServer: serve_forever
                socketserver.BaseServer -> socketserver.BaseServer: _handle_request_noblock
                    socketserver.BaseServer -> werkzeug.BaseWSGIServer: get_request
                    note left: override
                    socketserver.BaseServer -> socketserver.BaseServer: verify_request
                    socketserver.BaseServer -> socketserver.BaseServer: process_request
                        socketserver.BaseServer -> socketserver.BaseServer: finish_request
                            socketserver.BaseServer -> werkzeug.WSGIRequestHandler: __init__
                            note left: RequestHandlerClass
                            werkzeug.WSGIRequestHandler --> http.BaseHTTPRequestHandler: __init__
                            http.BaseHTTPRequestHandler --> socketserver.StreamRequestHandler: __init__
                            socketserver.StreamRequestHandler --> socketserver.BaseRequestHandler: __init__
                                socketserver.BaseRequestHandler -> socketserver.StreamRequestHandler: setup
                                'note left: override
                                socketserver.BaseRequestHandler -> werkzeug.WSGIRequestHandler: handle
                                note left: override
                                    werkzeug.WSGIRequestHandler -> http.BaseHTTPRequestHandler: handle
                                        http.BaseHTTPRequestHandler -> werkzeug.WSGIRequestHandler: handle_one_request
                                        note left: override
                                            werkzeug.WSGIRequestHandler -> werkzeug.WSGIRequestHandler: parse_request
                                            werkzeug.WSGIRequestHandler --> http.BaseHTTPRequestHandler: parse_request
                                            werkzeug.WSGIRequestHandler -> werkzeug.WSGIRequestHandler: run_wsgi
                                            note left: begin!
                                                werkzeug.WSGIRequestHandler -> werkzeug.WSGIRequestHandler: make_environ
                                                werkzeug.WSGIRequestHandler -> werkzeug.WSGIRequestHandler: run_wsgi.execute
                                                note left: app(environ, start_response)
                                socketserver.BaseRequestHandler -> socketserver.StreamRequestHandler: finish
                                'note left: override
            werkzeug.BaseWSGIServer -> werkzeug.BaseWSGIServer: server_close
@enduml
```
</div>
