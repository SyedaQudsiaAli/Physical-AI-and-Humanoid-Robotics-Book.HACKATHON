# Deploying on Vercel

You can deploy this Docusaurus project on Vercel with just a few steps.

## One-Click Deploy

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/SyedaQudsiaAli/Physical-AI-and-Humanoid-Robotics-Book.HACKATHON&project-name=Physical-AI-and-Humanoid-Robotics-Book&repository-name=Physical-AI-and-Humanoid-Robotics-Book)

## Manual Deployment

1. Install the Vercel CLI:
   ```bash
   npm i -g vercel
   ```

2. Navigate to the project directory:
   ```bash
   cd Physical-AI-and-Humanoid-Robotics-Book
   ```

3. Build the project:
   ```bash
   npm run build
   ```

4. Deploy to Vercel:
   ```bash
   vercel
   ```

## Environment Configuration

- Framework: Docusaurus
- Build Command: `cd .. && npm install && cd Physical-AI-and-Humanoid-Robotics-Book && npm run build`
- Output Directory: `Physical-AI-and-Humanoid-Robotics-Book/build`
- Install Command: `cd Physical-AI-and-Humanoid-Robotics-Book && npm install`

## Custom Domain

If you want to use a custom domain:
1. Go to your Vercel dashboard
2. Select your project
3. Go to Settings → Domains
4. Add your custom domain

For more information, visit [Vercel Documentation](https://vercel.com/docs).