FROM  public.ecr.aws/docker/library/node:18.14.2-alpine3.17

ARG REACT_APP_MOVIE_API_URL=http://ac1359cd0e3f44f8c91202d0c0671d17-1078869171.us-east-1.elb.amazonaws.com
ENV REACT_APP_MOVIE_API_URL=${REACT_APP_MOVIE_API_URL}

# Trigger Frontend CD (Second in sequence) - updated auth
WORKDIR /app

# Copy dependdency files
COPY package*.json ./

# Install packages
RUN npm ci

# Copy code
COPY . .

# Build the app
RUN npm run build

# Expose the port used by the React app
EXPOSE 3000

# Start the React app when the container starts
CMD ["npm", "run", "serve"]