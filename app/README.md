# MongoDB Express React NodeJS (MERN) Reference Implementation

This is an example implementation of a **restaurant review application** using the Express, Node.JS, React and MongoDB (MERN) stack.

### The following technologies are used:

* MongoDB Atlas for the database
* Express and Node.JS for the backend API
* React for the client

## Docker Setup

```bash
docker build -t xrootms/app-frontend:${GITHUB_SHA::7} --build-arg REACT_APP_API_URL=/api/v1 .
```
```bash
docker build -t xrootms/app-backend:latest .
```
```bash
docker push xrootms/app-frontend:${GITHUB_SHA::7}
```
```bash
docker push xrootms/app-backend:latest
```


