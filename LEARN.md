# Building Sveltekit and Docker Image

From: https://www.youtube.com/watch?v=GLQ3PXDgbIk

Used standard steps to create new sveltekit demo project with typescript
and tailwindcss.

    pnpm create svelte@latest sample
    cd sample
    pnpm i

Added node adapter package:

    pnpm i -D @sveltejs/adapter-node

In [svelte.config.js](svelte.config.js):

    //import adapter from '@sveltejs/adapter-auto';
    import adapter from '@sveltejs/adapter-node';

Created [.dockerignore](.dockerignore) to ignore node_modules.

Created [dockerfile](dockerfile) with build step:

    FROM node:lts-alpine as build
    WORKDIR /app
    COPY ./package*.json ./
    RUN npm install
    COPY . .
    RUN npm run build

    FROM node:lts-alpine as production
    COPY --from=build /app/build .
    COPY --from=build /app/package.json .
    COPY --from=build /app/package-lock.json .
    RUN npm ci --omit dev
    EXPOSE 3000
    CMD ["node", "."]

Build docker image:

    docker build . -t sample-image

Test docker image:

    docker run -p 3000:3000 --name sample-container sample-image

Now that it's been run for the first time, it can be started later with:

    docker start sample-image

