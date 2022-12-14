# README

Clone the following repository: https://github.com/mongodb-developer/pymongo-fastapi-crud
```bash
$ git clone https://github.com/mongodb-developer/pymongo-fastapi-crud
$ cd pymongo-fastapi-crud
```

## Dockerize the application
### Initialisation
First we need to add a '.env' file.
This file is used by the application to read environment variables.
```bash
$ touch .env
```
Then copy-paste the following lines into this file:
```bash
ATLAS_URI=${MONGODB_URI}
DB_NAME=${MONGODB_DB_NAME}
```

Replace the requirements.txt file with the following content:
```bash
fastapi[all]==0.88.0
pydantic==1.10.2
pymongo[srv]==4.3.3
pytest==7.2.0
python-dotenv==0.21.0
uvicorn==0.20.0
```

### Build the image
First create the Dockerfile
```bash
$ touch Dockerfile
```
Then copy-past the following content into this file:
```Dockerfile
FROM python:3.10-alpine

COPY requirements.txt /app/requirements.txt

RUN pip install --no-cache-dir --upgrade -r /app/requirements.txt

COPY .env /app/.env
COPY *.py /app/

WORKDIR /app

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```

And build the image:
```bash
docker build -t book_store .
```

So far so good, we have a book_store application ready to run...let connect it to its database !

## Connect the Database
First we'll need a working Mongodb instance.
Let's grab it from [DockerHub](https://hub.docker.com/_/mongo)
```bash
$ docker pull mongo
# verify our image is available locally
$ docker image ls
```

We will need both container to communicate with each other...
Let's create a dedicated network for both of them !
```bash
$ docker network create book_store
# verify it exists
$ docker network ls
```
Start the database
```bash
$ docker run --rm \
  --name book_db \
  --hostname book_db \
  --network book_store \
  mongo
```
Start the application
```bash
$  docker run --rm \
  -e MONGODB_URI='mongodb://book_db' \
  -e MONGODB_DB_NAME='book' \
  -p 8080:80 \
  --network book_store \
  book_store
```
Test everything work fine:
```bash
# Create a book
$ curl --location --request POST 'http://localhost:8080/book' \
  --header 'Content-Type: application/json' \
  --data-raw '{
    "title": "LotR",
    "author": "Tolkien",    
    "synopsis": "One ring to rule them all..."
  }'
# Fetch the list of created books
$ curl --location --request GET 'http://localhost:8080/book'
```

## Persist data
Stop your database container and recreate it
```bash
$ docker kill book_db
$ docker run --rm \
  --name book_db \
  --hostname book_db \
  --network book_store \
  mongo
```
Let's see if our books are still here:
```bash
# Fetch the list of created books
$ curl --location --request GET 'http://localhost:8080/book'
```
The response is empty :/
Let's create a volume to store our data and mount it into our database container:
```bash
# Stop the databse
$ docker kill book_db
# Create a volume to store our data
$ docker volume create book_data
# Recreate the database with our volume
$ docker run --rm \
  --name book_db \
  --hostname book_db \
  --network book_store \
  -v book_data:/data/db \
  mongo
```
Now create a Book, restart the database, and verify your data are not lost !