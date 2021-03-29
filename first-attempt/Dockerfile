# Get the Node.js base Docker image - shared!
FROM node:carbon
# Set the directory to run our Docker commands in
WORKDIR /app
# Copy your application source to this directory
COPY . .
# Start your application
CMD [ "node", "index.js" ]