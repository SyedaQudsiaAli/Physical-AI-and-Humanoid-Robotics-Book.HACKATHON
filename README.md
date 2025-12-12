# Physical AI and Humanoid Robotics Book

This repository contains the source code for the "Physical AI and Humanoid Robotics" book website, built with [Docusaurus](https://docusaurus.io/), a modern static website generator.

## 📚 Overview

This project hosts an educational resource focusing on the intersection of artificial intelligence and humanoid robotics. The content is structured as an online book/web resource that explores cutting-edge research, implementations, and theoretical foundations of physical AI and humanoid robotics.

## 🏗️ Project Structure

```
├── Physical-AI-and-Humanoid-Robotics-Book/     # Main Docusaurus website
│   ├── blog/                                  # Blog posts
│   ├── docs/                                  # Book chapters and documentation
│   ├── src/                                   # Source code and custom components
│   ├── static/                                # Static assets
│   ├── package.json                           # Dependencies and scripts
│   └── docusaurus.config.js                   # Site configuration
├── .github/                                   # GitHub workflows and configurations
├── .specify/                                  # Specification and project management tools
├── history/                                   # Historical prompts and records
├── specs/                                     # Technical specifications
├── CLAUDE.md                                  # Claude AI assistant rules and guidelines
└── README.md                                  # This file
```

## 🚀 Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) (version 16 or higher)
- [Yarn](https://yarnpkg.com/) or npm package manager

## 🌐 Deployment

### GitHub Pages
The site is automatically deployed to GitHub Pages using GitHub Actions.
The site will be available at: `https://SyedaQudsiaAli.github.io/Physical-AI-and-Humanoid-Robotics-Book.HACKATHON/`

**Important**: You need to enable GitHub Pages in your repository settings:
1. Go to your GitHub repository
2. Click on "Settings" tab
3. Scroll down to "Pages" section in the left sidebar
4. Under "Source", select "Deploy from a branch"
5. Select "gh-pages" as the branch and "/" as the folder
6. Click "Save"

After enabling GitHub Pages and once the GitHub Actions workflow completes successfully, your site will be live.

### Vercel
The site can also be deployed on Vercel for faster global loading.
[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/SyedaQudsiaAli/Physical-AI-and-Humanoid-Robotics-Book.HACKATHON&project-name=Physical-AI-and-Humanoid-Robotics-Book&repository-name=Physical-AI-and-Humanoid-Robotics-Book)

To deploy manually to Vercel:
1. Install the Vercel CLI: `npm i -g vercel`
2. Navigate to the project directory: `cd Physical-AI-and-Humanoid-Robotics-Book`
3. Run: `vercel --build-env DEPLOYMENT_PLATFORM=vercel`

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/SyedaQudsiaAli/Physical-AI-and-Humanoid-Robotics-Book.HACKATHON.git
   cd Physical-AI-and-Humanoid-Robotics-Book
   ```

2. Navigate to the main website directory:
   ```bash
   cd Physical-AI-and-Humanoid-Robotics-Book
   ```

3. Install dependencies:
   ```bash
   yarn install
   ```

### Local Development

```bash
# Start local development server
yarn start

# Open http://localhost:3000 to view the website
```

Most changes are reflected live without restarting the server.

### Building for Production

```bash
# Build static content for production
yarn build
```

This command generates static content into the `build` directory and can be served using any static content hosting service.

### Deployment

The site is automatically deployed to GitHub Pages using GitHub Actions when changes are pushed to the main branch.

The site will be available at: `https://SyedaQudsiaAli.github.io/Physical-AI-and-Humanoid-Robotics-Book.HACKATHON/`

To manually deploy, you can run:
```bash
cd Physical-AI-and-Humanoid-Robotics-Book
npm run build
```
Then push the build folder manually to the `gh-pages` branch.

## 🛠️ Tech Stack

- **Framework**: [Docusaurus](https://docusaurus.io/) v3+
- **Language**: JavaScript/TypeScript
- **Package Manager**: Yarn
- **Documentation**: Markdown/M DX
- **Deployment**: GitHub Pages

## 🤝 Contributing

We welcome contributions to improve the content and functionality of this educational resource. To contribute:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Commit your changes (`git commit -m 'Add amazing feature'`)
5. Push to the branch (`git push origin feature/amazing-feature`)
6. Open a Pull Request

### Content Contribution Guidelines

- Follow the existing document structure in the `docs/` directory
- Use clear, concise, and technically accurate language
- Include relevant examples and diagrams where appropriate
- Maintain consistent formatting and style

## 📋 Specification Framework

This project utilizes a specification-driven development (SDD) approach:

- **Specifications** (`specs/`): Detailed requirements and feature specifications
- **Prompts History** (`history/prompts/`): Complete record of all development prompts
- **Architecture Decision Records** (`history/adr/`): Important architectural decisions
- **Memory Constitution** (`.specify/memory/constitution.md`): Core project principles

## 🔒 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 📞 Contact

For questions, suggestions, or collaboration opportunities, please open an issue in this repository.

## 🙏 Acknowledgments

We thank all contributors who have helped develop this educational resource on Physical AI and Humanoid Robotics. This project serves the research community by making complex topics more accessible and understandable.