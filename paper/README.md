# Paper Reference

## Citation

> **CSAR: Containerized System Architecture for Robotics**  
> Gregorio Ambrosio-Cestero, Cipriano Galindo Andrades, Jose-Raul Ruiz-Sarmiento, Javier Gonzalez-Jimenez  
> MAPIR Group, Dept. of System Engineering and Automation  
> Málaga Institute for Mechatronics Engineering and Cyber-Physical Systems (IMECH.UMA)  
> University of Málaga, Spain

_DOI and journal reference will be added upon publication._

For BibTeX, see [`CITATION.cff`](../CITATION.cff) or use the "Cite this repository" button on GitHub.

---

## Abstract

Robotic applications increasingly rely on distributed computational infrastructures that combine embedded devices, edge servers, and cloud resources. This evolution, together with the collaborative nature of robotics projects, has made the development, integration, deployment, and long-term operation of robotic systems significantly more complex. In practice, multi-user robotics software teams face persistent challenges related to dependency isolation, compatibility, reproducibility, efficient sharing of specialized hardware, and deployment across heterogeneous environments.

In this paper, we present CSAR (Containerized System Architecture for Robotics), a container-centric architectural framework designed specifically for robotics teams and the edge–cloud continuum. CSAR combines LXC/LXD-based system containerization, ROS 2/DDS-based communication, and a three-layer edge infrastructure to organize computation into hardware-affine, persistent execution environments that remain decoupled from the volatility of experimental workloads. Through its Infrastructure Core, Platform and Multi-User Orchestration, and Compute and Acceleration layers, CSAR provides strong isolation, controlled resource sharing, and topology-aware networking for distributed robotic applications.

To demonstrate its validity, we describe a real deployment of CSAR in an academic robotics laboratory and evaluate it through representative use cases involving edge-offloaded 3D SLAM and GPU-accelerated semantic mapping. The results indicate that CSAR simplifies software integration, improves the utilization of shared computational resources, and facilitates safe prototyping, as well as reproducible and collaborative experimentation in robotics teams.

---

## Keywords

distributed robotics architecture · containerization · multi-user infrastructure · LXC/LXD · ROS 2 · DDS · edge–cloud continuum · resource isolation · reproducibility · GPU sharing
