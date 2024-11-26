[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/qvMX04VC)
# Assignment 10: MLFlow

This assignment is designed to help you understand machine learning operations using cloud primitives in a little more depth. We'll use MLflow as a ML model tracking server, and customize our deployment so that we take advantage of persistent cloud resources.

In part 1, you'll first create a local MLflow tracking server and start to understand how MLflow operates.

In Part 2, you'll connect your tracking server to cloud object storage, specifcally a DigitalOcean spaces bucket.

In Part 3, you'll first set up a local PostgreSQL container to run as the backing store for your MLflow deployment, and then you'll create a managed PostgreSQL database in DigitalOcean. 

To complete the assignment, you will need to:
- Deploy the MLflow tracking server on a droplet, with the Spaces bucket as your model artifact registry and the managed database as your backing server
- Apply firewall rules so that your tracking server droplet can only be accessed from within the VPC.

In D2L, please include the following:
- The name of your droplet (we will inspect firewall rules and connect to it from within the VPC for grading)
- The name of your Spaces bucket
- The name of your managed Postgres database
