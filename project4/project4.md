**Step 1: Install Nodejs**
  - Commands:
    - sudo apt install nodejs -y

**Step 2: Install MongoDB**
  - Commands:
    - sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6
    - echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu trusty/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list
    - sudo apt install -y mongodb
    - sudo systemctl start mongodb
    - sudo systemctl status mongodb
    - sudo apt install npm -y (Install npm)
    - sudo npm install body-parser (Process JSON requests)
    - Create necessary directories and edit files as required

**Step 3: Install Express and Setup Routes to Server**
  - Commands:
    - sudo npm install express mongoose
    - Create directories and edit files

**Step 4: Access the Routes with AngularJS**
  - Commands:
    - Edit files as required
    - node server.js (Start up server)