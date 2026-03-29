# Frontend App — Node.js Static Server

A clean portfolio frontend served by a minimal Node.js/Express server, fully Dockerized.

## Project Structure
```
frontend-app/
├── public/
│   └── index.html      ← The frontend (HTML/CSS/JS)
├── server.js           ← Express static file server
├── package.json
├── Dockerfile
└── .dockerignore
```

## Run with Docker

```bash
# Build the image
docker build -t frontend-app .

# Run the container
docker run -p 3000:3000 frontend-app
```

Open http://localhost:3000

## Run without Docker

```bash
npm install
node server.js
```

## Environment Variables

| Variable | Default | Description        |
|----------|---------|--------------------|
| PORT     | 3000    | Port to listen on  |

## Customizing

- Edit `public/index.html` to change the frontend
- Add more static assets (CSS, JS, images) to `public/`
- `server.js` serves everything in `public/` automatically
