---
title: "Contact"
description: "Contact Page"

cascade:
  showDate: false
  showAuthor: false
  invertPagination: true
---

{{< lead >}}
Frustrated because your robotics simulation seems to disobey the laws of physics?
Struggling with that URDF where the elbow link insists on being the shoulder link?
Trying to build clean TF Trees and ROS Nodes?
Or just want your robots to move with a bit more harmony and elegance?

Drop an email over to: jasmeet0915@gmail.com

Prefer chatting? Hit me up on:
<div class="contact-icons">
<a href="https://www.linkedin.com/in/jasmeetsingh2911/" target="_blank" title="LinkedIn">
    {{< icon "linkedin" >}}
</a>
</div>
{{< /lead >}}

<style>
  .contact-icons {
    display: flex;
    gap: 16px;
    justify-content: left;
    align-items: left;
    margin-top: 1.5rem;
  }

  .contact-icons a svg {
    width: 32px;
    height: 32px;
    fill: var(--color-text);
    transition: transform 0.2s ease, fill 0.3s ease, opacity 0.2s ease;
    opacity: 0.9;
  }

  .contact-icons a:hover svg {
    fill: var(--color-accent);
    transform: scale(1.15);
    opacity: 1;
  }

  .contact-icons a:active svg {
    transform: scale(0.95);
    opacity: 0.8;
  }
</style>
