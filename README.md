const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : { apiKey: "YOUR_API_KEY", authDomain: "YOUR_AUTH_DOMAIN", projectId: "YOUR_PROJECT_ID" /* ... other config */ };
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-kanban-app';
