"""

from http.server import HTTPServer, BaseHTTPRequestHandler
import os
from datetime import datetime

SAVE_DIR = "es_requests"
os.makedirs(SAVE_DIR, exist_ok=True)

class SimpleHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        self._handle_request()

    def do_PUT(self):
        self._handle_request()

    def _handle_request(self):
        # Get content length
        content_length = int(self.headers.get('Content-Length', 0))
        # Read request body
        body = self.rfile.read(content_length) if content_length else b''
        # Generate unique filename
        timestamp = datetime.now().strftime("%Y%m%d-%H%M%S-%f")
        filename = f"{SAVE_DIR}/es_{timestamp}_{self.command}.json"
        with open(filename, 'wb') as f:
            f.write(body)
        # Send response
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        response = f'{{"status": "saved", "filename": "{filename}"}}'
        self.wfile.write(response.encode())

if __name__ == '__main__':
    httpd = HTTPServer(('0.0.0.0', 9200), SimpleHandler)
    print("Listening on port 9200...")
    httpd.serve_forever()


"""
