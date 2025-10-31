"""

~~~


import argparse
import base64
import logging
import socket
from datetime import datetime, timezone
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from typing import Tuple


def _now_iso() -> str:
    return datetime.now(timezone.utc).astimezone().isoformat()


def _read_body(handler: BaseHTTPRequestHandler) -> bytes:
    # Support Content-Length and basic chunked transfer encoding.
    te = handler.headers.get("Transfer-Encoding", "").lower()
    if "chunked" in te:
        body = bytearray()
        r = handler.rfile
        while True:
            line = r.readline().strip()
            if not line:
                break
            try:
                chunk_size = int(line.split(b";", 1)[0], 16)
            except ValueError:
                break
            if chunk_size == 0:
                # consume trailer headers (if any) until blank line
                while True:
                    trailer = r.readline()
                    if trailer in (b"\r\n", b"\n", b""):
                        break
                break
            chunk = r.read(chunk_size)
            body.extend(chunk)
            # consume trailing CRLF
            r.read(2)
        return bytes(body)

    length_hdr = handler.headers.get("Content-Length")
    if not length_hdr:
        return b""
    try:
        length = int(length_hdr)
    except (TypeError, ValueError):
        length = 0
    if length <= 0:
        return b""
    return handler.rfile.read(length)


def _format_log_entry(handler: BaseHTTPRequestHandler, body: bytes) -> str:
    ts = _now_iso()
    client_ip, client_port = handler.client_address
    request_line = f"{handler.command} {handler.path} {handler.request_version}"
    # Headers as raw string
    headers_str = "".join(f"{k}: {v}\n" for k, v in handler.headers.items())
    body_b64 = base64.b64encode(body).decode("ascii") if body else ""
    body_len = len(body)
    entry = (
        f"-----\n"
        f"timestamp: {ts}\n"
        f"remote: {client_ip}:{client_port}\n"
        f"request_line: {request_line}\n"
        f"headers:\n{headers_str}"
        f"body_length: {body_len}\n"
        f"body_base64: {body_b64}\n"
    )
    return entry


class CaptureHandler(BaseHTTPRequestHandler):
    server_version = "MockCapture/1.0"

    def _capture_and_respond(self):
        body = _read_body(self)
        logging.info(_format_log_entry(self, body))

        # Respond with a simple 200 OK echo with request info
        content = (
            f"Captured {self.command} {self.path}\n"
            f"Time: {_now_iso()}\n"
            f"Bytes: {len(body)}\n"
        ).encode("utf-8")

        # For HEAD, do not send a body
        send_body = self.command.upper() != "HEAD"

        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(content) if send_body else 0))
        self.end_headers()
        if send_body:
            self.wfile.write(content)

    def do_GET(self):
        self._capture_and_respond()

    def do_POST(self):
        self._capture_and_respond()

    def do_PUT(self):
        self._capture_and_respond()

    def do_DELETE(self):
        self._capture_and_respond()

    def do_PATCH(self):
        self._capture_and_respond()

    def do_OPTIONS(self):
        self._capture_and_respond()

    def do_HEAD(self):
        self._capture_and_respond()

    def log_message(self, format: str, *args):
        # Also write access logs to the same logger for convenience
        logging.info("access - " + (format % args))


def _resolve_bind(bind: str) -> tuple[str, int]:
    host, _, port_str = bind.partition(":")
    if not port_str:
        raise ValueError("--bind must be in HOST:PORT format, or use --port")
    try:
        port = int(port_str)
    except ValueError:
        raise ValueError("Invalid port in --bind")
    return host or "0.0.0.0", port


def main():
    parser = argparse.ArgumentParser(description="Simple HTTP capture server")
    parser.add_argument("--port", type=int, default=8080, help="TCP port to listen on")
    parser.add_argument(
        "--bind",
        type=str,
        default=None,
        help="Optional bind address as HOST:PORT (overrides --port)",
    )
    parser.add_argument(
        "--log-file",
        type=str,
        default="requests.log",
        help="Path to append captured requests",
    )
    args = parser.parse_args()

    if args.bind:
        host, port = _resolve_bind(args.bind)
    else:
        host, port = "0.0.0.0", args.port

    # Configure logging to file (thread-safe)
    logging.basicConfig(
        level=logging.INFO,
        format="%(message)s",
        handlers=[logging.FileHandler(args.log_file, encoding="utf-8"), logging.StreamHandler()]
    )

    # Try binding early to provide clear error if port is in use
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        try:
            s.bind((host, port))
        except OSError as e:
            raise SystemExit(f"Failed to bind {host}:{port} - {e}")

    server = ThreadingHTTPServer((host, port), CaptureHandler)
    print(f"Listening on http://{host}:{port} — logging to {args.log_file}")
    try:
        server.serve_forever(poll_interval=0.5)
    except KeyboardInterrupt:
        print("\nShutting down...")
    finally:
        server.server_close()


if __name__ == "__main__":
    main()




~~~

"""
