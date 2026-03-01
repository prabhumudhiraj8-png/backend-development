
Project: Cloud-Native Search & Analytics Platform
(Mini Version of a Search Engineer System)

This aligns PERFECTLY with the Search Engineer role.

User → API Gateway → Document Service → Kafka → Index Service → Elasticsearch
                                  ↓
                             PostgreSQL

cloud-search-platform/
│
├── document_service/
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   ├── kafka_producer.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── indexing_service/
│   ├── main.py
│   ├── kafka_consumer.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── docker-compose.yml
└── README.md
🔹 Step 1: Document Service
document_service/requirements.txt
fastapi
uvicorn
psycopg2-binary
sqlalchemy
kafka-python
document_service/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "postgresql://user:password@db:5432/docdb"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)
Base = declarative_base()
document_service/models.py
from sqlalchemy import Column, Integer, String
from database import Base

class Document(Base):
    __tablename__ = "documents"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String)
    content = Column(String)
document_service/kafka_producer.py
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers='kafka:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def publish_event(document):
    producer.send("document_topic", document)
document_service/main.py
from fastapi import FastAPI
from database import SessionLocal, engine
from models import Base, Document
from kafka_producer import publish_event

Base.metadata.create_all(bind=engine)

app = FastAPI()

@app.post("/documents/")
def create_document(title: str, content: str):
    db = SessionLocal()
    doc = Document(title=title, content=content)
    db.add(doc)
    db.commit()
    db.refresh(doc)

    publish_event({"id": doc.id, "title": title, "content": content})

    return {"message": "Document created", "id": doc.id}

@app.get("/health")
def health():
    return {"status": "running"}
🔹 Step 2: Indexing Service
indexing_service/requirements.txt
kafka-python
indexing_service/kafka_consumer.py
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'document_topic',
    bootstrap_servers='kafka:9092',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    document = message.value
    print(f"Indexing document: {document}")
indexing_service/main.py
from kafka_consumer import consumer
🔹 Step 3: Dockerfile (Both Services)

Example for document_service/Dockerfile:

FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

Indexing service Dockerfile:

FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "main.py"]
🔹 Step 4: docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: docdb
    ports:
      - "5432:5432"

  zookeeper:
    image: wurstmeister/zookeeper

  kafka:
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_CREATE_TOPICS: "document_topic:1:1"

  document_service:
    build: ./document_service
    ports:
      - "8000:8000"
    depends_on:
      - db
      - kafka

  indexing_service:
    build: ./indexing_service
    depends_on:
      - kafka
