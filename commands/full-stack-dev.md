Start the full stack development environment.

1. Ensure PostgreSQL is running: `docker-compose up db -d`
2. Start backend: `cd backend && ./gradlew bootRun` (port 8080)
3. Start frontend: `cd frontend && npm run dev` (port 3000)

The frontend proxies API calls to the backend at localhost:8080.
