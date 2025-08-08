# School Converter

School Converter is a web application for managing and converting school data. It is built with Nuxt.

## Features

-   **School Data Management**: Manage a list of schools and their associated data.
-   **Data Conversion**: Convert school data based on defined thresholds.
-   **Extensible**: Easily add new school types and schools through a YAML configuration file.

## Usage

1.  Clone the repository.
2.  Install the dependencies: `npm install`
3.  Start the development server: `npm run dev`
4.  Open your browser to `http://localhost:3000` to view the application.

## Data Source

The school data is managed in the `app/data/schools.yaml` file. This file contains the different school types, their thresholds, and a list of schools with their abbreviations and cohort sizes.

## Setup

Make sure to install dependencies:

```bash
# npm
npm install

# pnpm
pnpm install

# yarn
yarn install

# bun
bun install
```

## Development Server

Start the development server on `http://localhost:3000`:

```bash
# npm
npm run dev

# pnpm
pnpm dev

# yarn
yarn dev

# bun
bun run dev
```

## Production

Build the application for production:

```bash
# npm
npm run build

# pnpm
pnpm build

# yarn
yarn build

# bun
bun run build
```

Locally preview production build:

```bash
# npm
npm run preview

# pnpm
pnpm preview

# yarn
yarn preview

# bun
bun run preview
```

Check out the [deployment documentation](https://nuxt.com/docs/getting-started/deployment) for more information.

## Contributing

Contributions are welcome! Please feel free to submit a pull request or open an issue.

## License

This project is licensed under the MIT License.