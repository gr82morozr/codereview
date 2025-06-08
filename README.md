from http.server import HTTPServer, BaseHTTPRequestHandler
from datetime import datetime
import os

SAVE_DIR = "es_requests"
os.makedirs(SAVE_DIR, exist_ok=True)

class RawLoggingHTTPRequestHandler(BaseHTTPRequestHandler):
    def handle(self):
        # Read the raw request line (the very first line sent by client)
        try:
            raw_line = self.rfile.readline(65537)
        except Exception as e:
            raw_line = b''
        timestamp = datetime.now().strftime("%Y%m%d-%H%M%S-%f")
        filename = f"{SAVE_DIR}/raw_{timestamp}.log"
        with open(filename, 'wb') as f:
            f.write(raw_line)
            # Read up to 32KB of the rest of the request (to avoid endless read)
            try:
                rest = self.rfile.read(32768)
                f.write(rest)
            except Exception:
                pass
        print(f"[{timestamp}] Logged request to {filename}")
        # Try to continue normal HTTP handling, else just close
        try:
            super().handle()
        except Exception:
            pass

    def do_GET(self):  # handle normal GET
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'OK')
    def do_POST(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'OK')
    def do_PUT(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'OK')
    def do_OPTIONS(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'OK')
    def log_message(self, format, *args):
        pass  # silence default logging

if __name__ == '__main__':
    print("Listening for *any* HTTP data on port 9200 ...")
    httpd = HTTPServer(('0.0.0.0', 9200), RawLoggingHTTPRequestHandler)
    httpd.serve_forever()
