**Step 1: Backend Configuration**
  - Commands:
    - curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -, sudo apt-get install nodejs (Update ubuntu sources list and install nodejs)
    - npm install express (Install Express.js using npm (Node Package Manager))
    - node index.js (Start server)
    - npm install mongoose (node package for working with mongodb)
    - vim .env (Edit .env file to contain DB authentication url)
![](db.jpg)
**Step 2: Frontend Creation**
  - Commands:
    - npx create_react_app client
    - npm install concurrently --save-dev (For running more than one commmands concurrently)
    - npm install nodemon --save-dev (To run and monitor server)
    - npm run dev (Run app)
![](todo.jpg)