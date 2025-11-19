# Static Hosting Setup Guide

This guide will help you deploy the TipJar static HTML version to various hosting platforms.

## Quick Setup (5 minutes)

### Step 1: Prepare Your Repository

If you haven't already, initialize your repository:

```bash
git init
git add .
git commit -m "Add TipJar static HTML version"
```

### Step 2: Choose a Hosting Platform

You can deploy to any of these platforms:

#### Option A: GitHub Pages
1. Create a new repository on GitHub
2. Add it as a remote: `git remote add origin <your-repo-url>`
3. Push your code: `git push -u origin main`
4. Go to repository Settings → Pages
5. Select branch: `main`, folder: `/docs`
6. Click Save

Your site will be at: `https://[username].github.io/[repo-name]/`

#### Option B: Netlify
1. Drag and drop the `docs/` folder to netlify.com
2. Or connect your Git repository
3. Set publish directory to `docs`

#### Option C: Vercel
1. Import your Git repository at vercel.com
2. Set output directory to `docs`
3. Deploy

### Step 3: Wait for Deployment

- Most platforms deploy in 1-3 minutes
- You'll receive a URL when ready

### Step 4: Access Your Site

Your site will be available at the URL provided by your hosting platform

## Troubleshooting

### Site Not Loading?

1. **Check your hosting platform's deployment logs**
2. **Wait a few minutes** - initial deployment can take up to 10 minutes
3. **Clear your browser cache** and try again
4. **Check the URL** - make sure you're using the correct format

### 404 Error?

1. Verify that `index.html` exists in the `docs/` folder
2. Check your hosting platform's configuration
3. Ensure the publish directory is set correctly

### OCR Not Working?

1. Make sure you're using HTTPS (not HTTP)
2. Try a different browser (Chrome/Edge recommended)
3. Check browser console for errors (F12)
4. Ensure the image is clear and high quality

## Custom Domain (Optional)

To use a custom domain like `tipjar.yourdomain.com`:

1. Configure DNS settings with your domain provider
2. Update your hosting platform's settings to use your custom domain
3. Consult your hosting platform's documentation for specific instructions

## Updating Your Site

To update your site after making changes:

```bash
git add docs/
git commit -m "Update TipJar"
git push
```

Your hosting platform will automatically redeploy your site within a few minutes.

## Features of This Static Version

✅ **No Build Process** - Just HTML, CSS, and JavaScript
✅ **No Backend Required** - Everything runs in the browser
✅ **Free Hosting** - Deploy to any static hosting platform
✅ **HTTPS Enabled** - Secure by default
✅ **Fast Loading** - Optimized for performance
✅ **Mobile Friendly** - Responsive design
✅ **Offline Capable** - Works after initial load

## File Structure

```
docs/
├── index.html          # Main application
├── app.js             # JavaScript logic
├── README.md          # Documentation
├── sample-report.md   # Example report formats
└── .nojekyll         # Prevents Jekyll processing
```

## Need Help?

- Check the [docs/README.md](docs/README.md) for usage instructions
- Review [sample-report.md](docs/sample-report.md) for OCR tips
- Consult your hosting platform's documentation

## Next Steps

1. ✅ Deploy to your chosen hosting platform
2. 📱 Test on mobile devices
3. 🎨 Customize colors and branding
4. 📊 Test with your actual reports
5. 🔗 Share the URL with your team

Enjoy using TipJar! 🎉
