---
title: "Otinus Polska sp. z o.o."
date: 2023-08-01
---
**Full-Stack Developer / DevOps Engineer**

At Otinus Polska, I was responsible for developing and maintaining internal systems that supported the daily work of the company. My role combined business application development, process automation, work with product data and infrastructure development for running and deploying applications.

On the development side, I worked mainly with applications written in PHP and JavaScript. I developed new features for internal systems, improved existing mechanisms, debugged problems and adapted solutions to real business processes in the company. One important area was product data management, including the analysis of technical and marketing data and the concept of data flow between ERP, PIM and marketplace systems.

A large part of my work was connected with developing and maintaining an application responsible for generating sales offers for customers in PDF format. The application was written in PHP and JavaScript and made it possible to create offers with products, add-ons, prices, discounts and other sales information.

As part of this process, products could have their own brochures prepared with Jinja2 templates. The brochure was then generated as a PDF file and added to the main sales offer document. The main offer included detailed sales information, such as the number of add-ons, their types, prices, discounts and other data needed by the customer. The documents were generated and combined using TCPDF and FPDI.

I also worked on problems related to rendering HTML to PDF, pagination of dynamic tables, footers, margins, page numbers, merging existing PDF files with dynamically generated pages and debugging generated HTML before PDF conversion.

Later, I added a new solution based on Puppeteer and Browsershot. It allowed the application to generate PDFs from HTML in a more flexible way. At the same time, I kept backward compatibility with the existing TCPDF-based system, so the older offer generation mechanisms still worked correctly.

At the same time, I worked on infrastructure and deployment. On the company server, I created a Kubernetes cluster based on k3s, which was later used to run company applications. I moved existing PHP and JavaScript applications to a containerized environment by preparing Docker images and the configuration needed to run them in Kubernetes.

In the next stage, I implemented application deployment in a GitOps model using FluxCD. Thanks to this, application and environment configuration could be managed from a Git repository, and changes were automatically synchronized with the Kubernetes cluster. This organized the deployment process, made environment configuration easier to control and reduced the risk of errors caused by manual deployment.

**Main areas of responsibility**

* Developing and maintaining internal business applications
* Programming applications in PHP and JavaScript
* Creating, improving and debugging features supporting company processes
* Developing an application for generating sales offers in PDF format
* Generating product brochures based on Jinja2 templates
* Adding generated brochure PDFs to main sales offer documents
* Rendering HTML to PDF using Puppeteer and Browsershot
* Working with TCPDF and FPDI to generate, build and merge PDF documents
* Keeping backward compatibility with the existing PDF generation system
* Solving problems with pagination, footers, margins and page numbers in PDF documents
* Debugging generated HTML before PDF conversion
* Analyzing and modeling product data
* Working on the concept of data flow between ERP, PIM and marketplace systems
* Separating technical data, marketing data, prices, descriptions and product images
* Containerizing PHP and JavaScript applications using Docker
* Creating a company Kubernetes cluster based on k3s
* Migrating company applications to the Kubernetes environment
* Implementing application deployment in the GitOps model
* Configuring FluxCD for automatic synchronization of changes from a Git repository
* Analyzing pods, services, logs and environment configuration
* Diagnosing problems related to containers, deployment and Kubernetes configuration

**Technology stack**

* Backend: PHP
* Frontend: JavaScript, HTML, CSS
* PDF: TCPDF, FPDI, Jinja2, Puppeteer, Browsershot
* DevOps / infrastructure: Docker, Kubernetes, k3s, FluxCD, GitOps
* Product data: ERP, PIM, marketplaces, data modeling
* Tools and processes: Git, repository-based deployment, debugging, deployment automation
* Scripting: Bash, Python

