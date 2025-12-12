# Deploying on Vercel

You can deploy this Docusaurus project on Vercel with just a few steps.

## One-Click Deploy

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/SyedaQudsiaAli/Physical-AI-and-Humanoid-Robotics-Book.HACKATHON)

## Manual Deployment

1. Install the Vercel CLI:
   ```bash
   npm i -g vercel
   ```

2. Run from the root of the repository:
   ```bash
   vercel --build-env DEPLOYMENT_PLATFORM=vercel
   ```

## Vercel Dashboard Deployment Configuration

If deploying via the Vercel dashboard:

- **Build Command**: `cd Physical-AI-and-Humanoid-Robotics-Book && npm install && npm run build`
- **Output Directory**: `Physical-AI-and-Humanoid-Robotics-Book/build`
- **Root Directory**: Select the root of the repository
- **Environment Variables**:
  - `DEPLOYMENT_PLATFORM`: `vercel`

## Environment Configuration

When Vercel builds the project, it will:
1. Install dependencies in the Physical-AI-and-Humanoid-Robotics-Book directory
2. Build the Docusaurus site with the appropriate configuration for Vercel deployment
3. Serve the built files from the correct directory

## Custom Domain

If you want to use a custom domain:
1. Go to your Vercel dashboard
2. Select your project
3. Go to Settings → Domains
4. Add your custom domain

For more information, visit [Vercel Documentation](https://vercel.com/docs).