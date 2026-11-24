FROM node:22-alpine

RUN corepack enable
WORKDIR /app

# pnpm-lock.yaml is the maintained lockfile (package-lock.json is stale and fails `npm ci`).
# pnpm 10+ refuses unapproved dependency build scripts (sharp, swc, ...), so allow them here.
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --config.dangerously-allow-all-builds=true

COPY . .
RUN pnpm build

ENV NODE_ENV=production
EXPOSE 3000
CMD ["pnpm", "start", "-H", "0.0.0.0"]
