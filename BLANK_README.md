<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->
<a name="readme-top"></a>
<!--
*** Thanks for checking out the Best-README-Template. If you have a suggestion
*** that would make this better, please fork the repo and create a pull request
*** or simply open an issue with the tag "enhancement".
*** Don't forget to give the project a star!
*** Thanks again! Now go create something AMAZING! :D
-->



<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->
[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]



<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/github_username/repo_name">
    <img src="images/logo.png" alt="Logo" width="80" height="80">
  </a>

<h3 align="center">Easy install of k3s using Proxmox and Ubuntu</h3>

  <p align="center">
    project_description
    <br />
    <a href="https://github.com/github_username/repo_name"><strong>Explore the docs »</strong></a>
    <br />
    <br />
    <a href="https://github.com/github_username/repo_name">View Demo</a>
    ·
    <a href="https://github.com/github_username/repo_name/issues">Report Bug</a>
    ·
    <a href="https://github.com/github_username/repo_name/issues">Request Feature</a>
  </p>
</div>



<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

[![Product Name Screen Shot][product-screenshot]](https://example.com)

This guide will hopefully get you up and running with a k3s cluster with relatively little effort.

Here's a blank template to get started: To avoid retyping too much info. Do a search and replace with your text editor for the following: `github_username`, `repo_name`, `twitter_handle`, `linkedin_username`, `email_client`, `email`, `project_title`, `project_description`

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- GETTING STARTED -->
## Getting Started

I base most of my current projects on Proxmox 7.2.x. Proxmox is easy to install and can run fine on relatively outdated hardware, and as it is based on Debian it is rather familiar to me.

### Prerequisites

* Required packages to build Cloud init templates
  ```sh
  apt install cloud-init libguestfs-tools
  ```
For this project I have use a readily available Cloud image of Ubuntu 22.04 LTS (Jammy)
```sh
  wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
  ```
This image works well for me, but feel free to give other images a spin

* Install qemu-guest-agent and nano on the Ubuntu image:
```sh
sudo virt-customize -a jammy-server-cloudimg-amd64.img --install qemu-guest-agent
sudo virt-customize -a jammy-server-cloudimg-amd64.img --install nano
sudo virt-customize -a jammy-server-cloudimg-amd64.img --install mc 
```
For lab/testing I don't bother too much with ssh keys, but enable enable PasswordAuthentication


```sh 
sudo virt-customize -a jammy-server-cloudimg-amd64.img --run-command "sed -i 's/.*PasswordAuthentication.*/PasswordAuthentication yes/g' /etc/ssh/sshd_config" 
```
Now the image should be ready for creating a reasonably good template. For the sake of it I use 2GB ram and 2vCPUs. Feel free to adjust as needed.
The image is only 2.2GB by default, thus I chose to add 13GB as I need this for Kubernetes

```sh 
sudo qm create 9000 --name "ubuntu-2204-cloudinit-template" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
sudo qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm [replace local-lvm if you have a different name for your datastore]
sudo qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
sudo qm set 9000 --boot c --bootdisk scsi0
sudo qm set 9000 --ide2 local-lvm:cloudinit
sudo qm set 9000 --serial0 socket --vga serial0
sudo qm set 9000 --agent enabled=1
sudo qm resize 9000 scsi0 +13G 
```
Now the image should be present in Proxmox, and you can edit a few items. For my purpose I add username and password in Cloud Init. I also add a VLAN tag under network properties.

When done you can convert the VM to a template

```sh 
sudo qm template 9000
```
### Getting VMs ready for k3s

When deploying a lab cluster I normally deploy a single control plane and two workers. For lager project I prefer to have three control plane nodes which give me a High Availability of the etcd database. For the sake of it I will deploy the latter in this example. I also add static IPs in my VLAN100

```sh 
sudo qm clone 9000 600 --name k3s-ctrl01
sudo qm set 600 --ipconfig0 ip=10.0.100.11/24,gw=10.0.100.1
sudo qm clone 9000 601 --name k3s-ctrl02
sudo qm set 601 --ipconfig0 ip=10.0.100.12/24,gw=10.0.100.1
sudo qm clone 9000 602 --name k3s-ctrl03
sudo qm set 602 --ipconfig0 ip=10.0.100.13/24,gw=10.0.100.1
sudo qm clone 9000 603 --name k3s-worker01
sudo qm set 603 --ipconfig0 ip=10.0.100.14/24,gw=10.0.100.1
sudo qm clone 9000 604 --name k3s-worker02
sudo qm set 604 --ipconfig0 ip=10.0.100.15/24,gw=10.0.100.1
sudo qm clone 9000 605 --name k3s-worker03
sudo qm set 600 --ipconfig0 ip=10.0.100.16/24,gw=10.0.100.1
```
If you need a full clone rather than a linked clone (liked clone will lock the template, eg you are not able to delete it) you will need to specify this when cloning

```sh
sudo qm clone 9000 600 --name k3s-ctrl01 --full
```
If done right you should have 6 fully working VMs ready for Kubernetes.

We'll start of with creating the first Control plane node on the first VM. The cluster will have a load balanced address of 10.0.100.10 (created later) and in this example a predefined token.

```sh
curl -sfL https://get.k3s.io | sh -s - server \
--token=78qrq0.1sr26zankoh22ou7 \
--tls-san k3s-demo.local --tls-san 10.0.100.10 \
--cluster-init
```

### Installation

1. Get a free API Key at [https://example.com](https://example.com)
2. Clone the repo
   ```sh
   git clone https://github.com/github_username/repo_name.git
   ```
3. Install NPM packages
   ```sh
   npm install
   ```
4. Enter your API in `config.js`
   ```js
   const API_KEY = 'ENTER YOUR API';
   ```

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- USAGE EXAMPLES -->
## Usage

Use this space to show useful examples of how a project can be used. Additional screenshots, code examples and demos work well in this space. You may also link to more resources.

_For more examples, please refer to the [Documentation](https://example.com)_

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ROADMAP -->
## Roadmap

- [ ] Feature 1
- [ ] Feature 2
- [ ] Feature 3
    - [ ] Nested Feature

See the [open issues](https://github.com/github_username/repo_name/issues) for a full list of proposed features (and known issues).

<p align="right">(<a href="#readme-top">back to top</a>)</p>






<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE.txt` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>






<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

* []()
* []()
* []()

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/github_username/repo_name.svg?style=for-the-badge
[contributors-url]: https://github.com/github_username/repo_name/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/github_username/repo_name.svg?style=for-the-badge
[forks-url]: https://github.com/github_username/repo_name/network/members
[stars-shield]: https://img.shields.io/github/stars/github_username/repo_name.svg?style=for-the-badge
[stars-url]: https://github.com/github_username/repo_name/stargazers
[issues-shield]: https://img.shields.io/github/issues/github_username/repo_name.svg?style=for-the-badge
[issues-url]: https://github.com/github_username/repo_name/issues
[license-shield]: https://img.shields.io/github/license/github_username/repo_name.svg?style=for-the-badge
[license-url]: https://github.com/github_username/repo_name/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/linkedin_username
[product-screenshot]: images/screenshot.png
[Next.js]: https://img.shields.io/badge/next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white
[Next-url]: https://nextjs.org/
[React.js]: https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[React-url]: https://reactjs.org/
[Vue.js]: https://img.shields.io/badge/Vue.js-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D
[Vue-url]: https://vuejs.org/
[Angular.io]: https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white
[Angular-url]: https://angular.io/
[Svelte.dev]: https://img.shields.io/badge/Svelte-4A4A55?style=for-the-badge&logo=svelte&logoColor=FF3E00
[Svelte-url]: https://svelte.dev/
[Laravel.com]: https://img.shields.io/badge/Laravel-FF2D20?style=for-the-badge&logo=laravel&logoColor=white
[Laravel-url]: https://laravel.com
[Bootstrap.com]: https://img.shields.io/badge/Bootstrap-563D7C?style=for-the-badge&logo=bootstrap&logoColor=white
[Bootstrap-url]: https://getbootstrap.com
[JQuery.com]: https://img.shields.io/badge/jQuery-0769AD?style=for-the-badge&logo=jquery&logoColor=white
[JQuery-url]: https://jquery.com 
