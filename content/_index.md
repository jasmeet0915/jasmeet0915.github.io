---
title: "Welcome to my website!"
description: "This page was built using the Blowfish theme for Hugo."
---

<style>
  /* Default layout – desktop and larger screens */
.home-buttons {
  display: inline-flex;
  gap: 12px;
}

/* Mobile adjustments */
@media (max-width: 600px) {
  .home-buttons {
    flex-direction: column;   /* Stack buttons vertically */
    width: 100%;              /* Take full width */
  }

  /* Make buttons easier to tap on phones */
  .home-buttons .button {
    width: 100%;              /* Full-width buttons */
    text-align: center;
    padding: 14px 18px;       /* Larger tap area */
    font-size: 1rem;          /* Slightly larger text for readability */
  }

  /* Extra spacing between stacked buttons */
  .home-buttons {
    gap: 16px;
  }
}
</style>

<div class="home-buttons">

  {{< button href="/files/Jasmeet_Singh_CV.pdf" target="_self" >}}
  Download CV ↓
  {{< /button >}}

  {{< button href="https://www.instructables.com/member/Jasmeeet%20Singh/" target="_self" >}}
  Instructables ↗
  {{< /button >}}

  {{< button href="https://github.com/atom-robotics-lab" target="_self" >}}
  A.T.O.M Robotics Lab ↗
  {{< /button >}}

</div>

{{< carousel images="projects_gallery/*" aspectRatio="16-9" interval="2500" >}}
