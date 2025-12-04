
FROM python:3.11-slim

# Create non-root user with uid 1001 (OpenShift likes arbitrary UIDs >= 100000, but 1001 works fine with random UID too)
RUN useradd -u 1001 -m flask

WORKDIR /app

# Install deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY . .

ENV FLASK_APP=app.py
ENV PORT=5000

# OpenShift runs as random UID; make sure dir is writable
RUN chown -R 1001 /app
USER 1001

EXPOSE 5000

CMD ["python", "app.py"]
