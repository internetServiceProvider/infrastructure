
# Infrastructure Documentation

---

This repository serves as a comprehensive documentation hub for our network infrastructure. It details the planning, implementation, and configuration of key components, including the Huawei OLT plans configuration, our monitoring stack (Grafana and Prometheus), and our FreeRADIUS server.


![image](https://github.com/user-attachments/assets/c7e80317-033b-441f-8093-be14e51a0920)

---

## Huawei OLT Implementation Plans

This section contains all the documentation related to the planning and execution of our Huawei OLT deployment. You'll find:

* **Configuration guides:** Step-by-step instructions for configuring the Huawei OLT, including port assignments, VLAN setups, and service provisioning.
* **Service plans:** Details on how different services (e.g., internet access, IPTV) are provisioned and managed on the OLT.
* **Troubleshooting guides:** Common issues and their resolutions specific to the Huawei OLT.

---

## Grafana and Prometheus Implementation

We've implemented a robust monitoring solution using Grafana and Prometheus to gain insights into our network's performance and health. This section outlines:

* **Architecture overview:** How Prometheus collects metrics and how Grafana visualizes them.
* **Prometheus configuration:** Details on scrape targets, retention policies, and alert rules.
* **Grafana dashboards:** Documentation for our custom Grafana dashboards, including data sources and visualization types.
* **Installation steps:** A guide to setting up Grafana and Prometheus from scratch.

---

## FreeRADIUS Implementation

FreeRADIUS is integral to our network for authentication, authorization, and accounting (AAA) services. This section covers:

* **Installation and configuration:** Step-by-step instructions for setting up FreeRADIUS.
* **Client configuration:** How to configure network devices (e.g., switches, routers, OLT) to authenticate against FreeRADIUS.
* **User management:** Documentation on adding, modifying, and managing user accounts within FreeRADIUS.
* **Policy enforcement:** How FreeRADIUS policies are used for access control and service limitations.
* **Logging and auditing:** Information on FreeRADIUS logs and how they are used for auditing and troubleshooting.
