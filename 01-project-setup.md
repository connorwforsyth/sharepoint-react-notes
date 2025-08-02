# Project Setup Guide

This guide walks through setting up a SharePoint Framework project with React, TypeScript, Tailwind CSS, and shadcn.

## Prerequisites

Before starting, ensure you have:

- **Node.js 18.x or later** - Download from [nodejs.org](https://nodejs.org/)
- **SharePoint Online tenant** - Office 365 developer subscription
- **Visual Studio Code** - Recommended IDE
- **Git** - For version control

## Step 1: Install SharePoint Framework Tools

Install the SharePoint Framework Yeoman generator globally:

```bash
npm install -g @microsoft/generator-sharepoint
npm install -g yo
npm install -g gulp-cli
```

## Step 2: Create New SPFx Project

Create a new directory and initialize the project:

```bash
mkdir my-spfx-app
cd my-spfx-app
yo @microsoft/sharepoint
```

### Configuration Options

When prompted, select these options:

- **Solution name**: `my-spfx-app`
- **Target environment**: `SharePoint Online only (latest)`
- **Place files**: `Use the current folder`
- **Deployment option**: `Tenant-scoped deployment`
- **Permissions**: `No`
- **Component type**: `WebPart`
- **Component name**: `MyApp`
- **Component description**: `Modern SPFx app with React`
- **Framework**: `React`

## Step 3: Project Structure

After generation, your project structure should look like:

```
my-spfx-app/
├── config/
│   ├── package-solution.json
│   ├── serve.json
│   └── write-manifests.json
├── src/
│   └── webparts/
│       └── myApp/
│           ├── MyAppWebPart.ts
│           └── components/
│               ├── MyApp.tsx
│               ├── MyApp.module.scss
│               └── IMyAppProps.ts
├── teams/
├── package.json
├── tsconfig.json
└── gulpfile.js
```

## Step 4: Install Additional Dependencies

Install required packages for our tech stack:

```bash
# Tailwind CSS
npm install -D tailwindcss postcss autoprefixer
npm install -D @tailwindcss/typography @tailwindcss/forms

# React Router
npm install react-router-dom
npm install -D @types/react-router-dom

# Additional utilities
npm install clsx tailwind-merge
npm install lucide-react

# Development tools
npm install -D @types/node
```

## Step 5: Initialize Tailwind CSS

Create Tailwind configuration:

```bash
npx tailwindcss init -p
```

This creates `tailwind.config.js` and `postcss.config.js` files.

## Step 6: Configure Build Pipeline

### Update gulpfile.js

Add PostCSS processing to the build pipeline:

```javascript
const build = require("@microsoft/sp-build-web");
const path = require("path");

// Add PostCSS processing
build.sass.setConfig({
  useCSSModules: false,
  sassMatch: [
    path.resolve(__dirname, "src/**/*.scss"),
    path.resolve(__dirname, "src/**/*.css"),
  ],
});

// Suppress TypeScript warnings for Node modules
build.configureWebpack.mergeConfig({
  additionalConfiguration: (generatedConfiguration) => {
    // Add PostCSS loader for CSS files
    const rules = generatedConfiguration.module.rules;

    // Find the CSS rule and modify it
    const cssRule = rules.find(
      (rule) => rule.test && rule.test.toString().includes("css")
    );

    if (cssRule) {
      cssRule.use = [
        ...cssRule.use,
        {
          loader: "postcss-loader",
          options: {
            postcssOptions: {
              plugins: [require("tailwindcss"), require("autoprefixer")],
            },
          },
        },
      ];
    }

    return generatedConfiguration;
  },
});

build.initialize(require("gulp"));
```

## Step 7: Environment Verification

Test your setup:

```bash
# Build the solution
gulp build

# Start development server
gulp serve

# Package for deployment
gulp bundle --ship
gulp package-solution --ship
```

## Step 8: Visual Studio Code Setup

### Recommended Extensions

Install these VS Code extensions:

- **SharePoint Framework Toolkit**
- **Tailwind CSS IntelliSense**
- **TypeScript Importer**
- **ES7+ React/Redux/React-Native snippets**
- **Prettier - Code formatter**
- **Auto Rename Tag**

### VS Code Settings

Create `.vscode/settings.json`:

```json
{
  "typescript.suggest.autoImports": true,
  "typescript.updateImportsOnFileMove.enabled": "always",
  "emmet.includeLanguages": {
    "typescript": "html",
    "typescriptreact": "html"
  },
  "tailwindCSS.includeLanguages": {
    "typescript": "html",
    "typescriptreact": "html"
  },
  "css.validate": false,
  "scss.validate": false,
  "files.associations": {
    "*.css": "tailwindcss"
  }
}
```

## Step 9: TypeScript Configuration

Update `tsconfig.json` for strict type checking:

```json
{
  "extends": "./node_modules/@microsoft/rush-stack-compiler-4.7/includes/tsconfig-web.json",
  "compilerOptions": {
    "target": "es2017",
    "forceConsistentCasingInFileNames": true,
    "module": "esnext",
    "moduleResolution": "node",
    "jsx": "react",
    "declaration": true,
    "sourceMap": true,
    "experimentalDecorators": true,
    "skipLibCheck": false,
    "inlineSources": false,
    "strict": true,
    "noImplicitReturns": true,
    "noImplicitAny": true,
    "baseUrl": "./src",
    "paths": {
      "@/*": ["*"],
      "@/components/*": ["webparts/*/components/*"],
      "@/types/*": ["types/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.tsx"],
  "exclude": ["node_modules", "lib"]
}
```

## Next Steps

1. Configure Tailwind CSS and shadcn ([03-tailwind-shadcn-setup.md](./03-tailwind-shadcn-setup.md))
2. Set up TypeScript patterns ([02-typescript-configuration.md](./02-typescript-configuration.md))
3. Implement React functional components ([04-react-functional-patterns.md](./04-react-functional-patterns.md))

## Troubleshooting

### Common Issues

**Node.js Version**: Ensure you're using Node.js 18.x. Use nvm to manage versions:

```bash
nvm install 18
nvm use 18
```

**Build Errors**: Clear node_modules and reinstall:

```bash
rm -rf node_modules package-lock.json
npm install
```

**HTTPS Certificate**: Trust the development certificate:

```bash
gulp trust-dev-cert
```

### Useful Commands

```bash
# Clean and rebuild
gulp clean && gulp build

# Serve with specific configuration
gulp serve --config serve.json

# Bundle for production
gulp bundle --ship && gulp package-solution --ship
```

## References

- [SharePoint Framework Overview](https://learn.microsoft.com/en-us/sharepoint/dev/spfx/sharepoint-framework-overview)
- [SPFx Web Parts Development](https://learn.microsoft.com/en-us/training/modules/sharepoint-spfx-web-parts/)
- [Node.js Requirements](https://learn.microsoft.com/en-us/sharepoint/dev/spfx/set-up-your-development-environment)
