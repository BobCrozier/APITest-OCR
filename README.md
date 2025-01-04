# APITest-OCR
Learning how to deploy an API

python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

pip install fastapi uvicorn httpx python-multipart

services:
  - name: ocr-api
    url: http://ocr-api:8000
    routes:
      - name: ocr-route
        paths:
          - /ocr
        strip_path: true

version: '3'
services:
  kong:
    image: kong:latest
    environment:
      - KONG_DATABASE=off
      - KONG_DECLARATIVE_CONFIG=/kong.yml
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
    ports:
      - "8000:8000"
    volumes:
      - ./kong.yml:/kong.yml:ro

  ocr-api:
    build: .
    environment:
      - OCR_SERVICE_URL=http://ocr-service:8081/process
    depends_on:
      - ocr-service

  ocr-service:
    image: your-ocr-service-image  # You'll need to specify your OCR service image
    ports:
      - "8081:8081"

FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

docker-compose up --build

curl -X POST http://localhost:8000/ocr/api/v1/process-document \
     -F "file=@your-document.pdf"

from fastapi import FastAPI, File, UploadFile
import httpx
import os
from pydantic import BaseModel
from typing import Dict

app = FastAPI()

# Configuration
OCR_SERVICE_URL = os.getenv("OCR_SERVICE_URL", "http://ocr-service:8081/process")

class OCRResponse(BaseModel):
    text: str
    metadata: Dict = {}

@app.post("/api/v1/process-document", response_model=OCRResponse)
async def process_document(file: UploadFile = File(...)):
    """
    Endpoint to process PDF documents and extract text using OCR
    """
    try:
        # Read the uploaded file
        content = await file.read()
        
        # Prepare the file for the OCR service
        files = {'file': (file.filename, content, 'application/pdf')}
        
        # Call the OCR microservice
        async with httpx.AsyncClient() as client:
            response = await client.post(
                OCR_SERVICE_URL,
                files=files
            )
            
            if response.status_code != 200:
                return {"error": "OCR service error", "details": response.text}
                
            ocr_result = response.json()
            
            return OCRResponse(
                text=ocr_result.get("text", ""),
                metadata={
                    "filename": file.filename,
                    "page_count": ocr_result.get("page_count", 0),
                    "processed_at": ocr_result.get("processed_at")
                }
            )
            
    except Exception as e:
        return {"error": str(e)}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
