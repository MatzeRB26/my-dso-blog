---
sidebar_position: 1
title: Docusaurus Blog Project
description: Documentation of the Docusaurus blog setup and custom configuration process.
---

# Docusaurus Blog Project Setup

This project documents the configuration and personalization process of the Docusaurus developer blog base template.

## Overview

The goal of this project was to take the standard developer blog template and customize its configuration, branding, dynamic URLs, and deployment pipeline for personal use.

## Configuration Steps Completed

### 1. Dynamic Repository Configuration
* Integrated `dotenv` to dynamically handle environment variables.
* Added a `gitRepositoryUrl` variable in `docusaurus.config.ts`.
* Updated the `editUrl` for docs and blog sections to reflect the dynamic repository link.

### 2. Branding and Navigation Customization
* **Navbar**:
  * Customized the site title and navigation items.
  * Updated the GitHub navbar item to use `gitRepositoryUrl`.
* **Footer**:
  * Added a direct link to the project overview (`/docs/projects/overview`) under the Docs section.
  * Removed the unnecessary Community column.
  * Added a reference link pointing to the original template repository in the More section.
  * Updated the copyright string to include personal attribution and the required template extension note.

### 3. Documentation & Repository Cleanup
* Simplified the `README.md` file by updating the deployment instructions to describe automated deployment via GitHub Actions.
* Removed outdated sections (e.g., Contributing) from the README.
* Configured GitHub Pages source settings to build and deploy via **GitHub Actions**.

## Deployment Pipeline

The project relies on GitHub Actions. Every push to the `main` branch triggers an automated build and deployment pipeline that publishes the static site directly to **GitHub Pages**.