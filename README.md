# Travel Story App

## Overview
The Travel Story App is a full-stack web application that allows users to create, share, and explore travel stories. Users can register, log in, add their travel experiences with images, search for stories, filter by date, and view stories on a homepage. The app consists of a React-based frontend for the user interface and a Node.js/Express backend for API services, with MongoDB as the database.

### Features
- User authentication (registration and login)
- Add, edit, delete, and view travel stories
- Image upload and management
- Search stories by title, content, or location
- Filter stories by date range
- Like stories
- Responsive UI built with React and Tailwind CSS

## Architecture
- **Frontend**: React application built with Vite, using React Router for navigation, Axios for API calls, and Tailwind CSS for styling.
- **Backend**: Node.js server using Express.js, with JWT for authentication, Multer for file uploads, and Mongoose for MongoDB integration.
- **Database**: MongoDB Atlas for storing user data and travel stories.
- **Deployment**: Configured for Kubernetes with Docker containers.

## Port Configuration
- **Backend Port**: The backend server runs on port 5000 by default (configurable via `PORT` environment variable). In the Kubernetes deployment, it is exposed on port 3000 inside the container.
- **Frontend Port**: The frontend development server runs on port 5173 (default Vite port), and the production build serves static files.

## Health Check
The application includes health check endpoints for monitoring in production environments:
- **Endpoint**: `/health`
- **Method**: GET
- **Purpose**: Used by Kubernetes readiness and liveness probes to ensure the application is running correctly.
- **Configuration**: In the Kubernetes deployment (`k8s/deployment.yaml`), readiness and liveness probes are set to check this endpoint every 10-30 seconds after an initial delay.
- **Note**: The `/health` endpoint is not explicitly implemented in the current backend code. It should return a simple JSON response (e.g., `{"status": "ok"}`) to indicate the service is healthy. If needed, add this route to `backend/index.js`.

## Getting Started
### Prerequisites
- Node.js (v14 or higher)
- MongoDB (local or Atlas)
- Docker (for containerized deployment)

### Installation
1. Clone the repository:
   ```
   git clone <repository-url>
   cd TravelStory
   ```

2. Set up the backend:
   ```
   cd backend
   npm install
   cp .env.example .env  # Configure environment variables (DB_URL, ACCESS_TOKEN_SECRET, etc.)
   npm run dev  # For development
   ```

3. Set up the frontend:
   ```
   cd ../frontend
   npm install
   npm run dev  # For development
   ```

4. For production, use Docker Compose:
   ```
   docker-compose up --build
   ```

### Environment Variables
- `DB_URL`: MongoDB connection string
- `ACCESS_TOKEN_SECRET`: JWT secret for authentication
- `PORT`: Backend port (default 5000)
- `BASE_URL`: Base URL for the application

## API Endpoints
- `POST /create-account`: Register a new user
- `POST /login`: User login
- `GET /get-user`: Get user details
- `POST /add-travel-story`: Add a new travel story
- `GET /get-all-stories`: Get all stories
- `PUT /edit-story/:id`: Edit a story
- `DELETE /delete-story/:id`: Delete a story
- `GET /search`: Search stories
- `GET /travel-stories/filter`: Filter stories by date
- And more...

## Deployment
The app is configured for Kubernetes deployment with the following components:
- **Deployment**: `k8s/deployment.yaml` - Defines the backend pod with probes
- **Service**: `k8s/service.yaml` - Exposes the backend service

Use `kubectl apply -f k8s/` to deploy to a Kubernetes cluster.

## Contributing
1. Fork the repository
2. Create a feature branch
3. Commit changes
4. Push to the branch
5. Create a Pull Request


