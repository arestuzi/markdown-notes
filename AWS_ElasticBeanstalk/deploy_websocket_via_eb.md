# How to deploy websocket on AWS ElasticBeanstalk?

## Environment

- Python 3.7 EB platform version 3.1.5
- Flask 1.1.2

## Solution

1. Save following code as application.py

    ```python
    from flask import Flask, render_template
    from flask_sockets import Sockets

    app = Flask(__name__)
    app.debug = True

    sockets = Sockets(app)

    @sockets.route('/echo')
    def echo_socket(ws):
        while True:
            message = ws.receive()
            ws.send("I am Server")

    @app.route('/')
    def hello():
        return 'Hello World!'

    @app.route('/echo_test', methods=['GET'])
    def echo_test():
        return render_template('echo_test.html')

    # if __name__ == '__main__':
    #    app.run()
    ```

2. Save following code as Procfile.

    ```txt
    web: gunicorn --bind :8000 --workers 3 --threads 2 application:app -k flask_sockets.worker
    ```

3. Save python packages dependencies to requirements.txt

    ```bash
    pip freeze > requirements.txt

    # the Flask version I am using
    cffi==1.14.4
    click==7.1.2
    Flask==1.1.2
    Flask-Sockets==0.2.1
    greenlet==1.0.0
    itsdangerous==1.1.0
    Jinja2==2.11.2
    MarkupSafe==1.1.1
    pycparser==2.20
    six==1.15.0
    websocket-client==0.57.0
    Werkzeug==1.0.1
    zope.event==4.5.0
    zope.interface==5.2.0
    ```

4. Compress files into *.zip

    ```bash
    zip -r mywebsocket.zip .
    ```

5. Upload the file to AWS EB environment.

6. run following code for testing.

    ```python
    from websocket import create_connection


    def client_handle():
        ws = create_connection('ws://EB_URL/echo')
        while True:
            if ws.connected:
                ws.send('hi,i am ws client')
                result = ws.recv()
                print(f"client received:{result}")
                ws.close()


    if __name__ == "__main__":
        client_handle()
    ```

7. Output

    ```bash
    python client.py
    client received:I am Server
    ```
