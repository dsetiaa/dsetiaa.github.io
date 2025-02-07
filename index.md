---
layout: page
title: About
permalink: /
---

<div class="about-wrapper">
  <div class="about-content">
    <p>Hi I'm Divyanshu. I'm a founding engineer at Tensorfuse, a YC-backed startup building infrastructure for scalable ML. I am primarily a cloud engineer with expertise in deploying with AWS Cloudformation. I also do some backend engineering.</p>

    <p>I am a computer science graduate from IIT Roorkee. I have some experience in developing mobile apps using flutter and have also some knowledge in classic cryptography</p>

    <p>In my free time I love solving puzzles, playing video games and watching the shows that my friends recommend to me.</p>

    <div class="about-image-container">
        <div class="about-image">
            <img src="/assets/images/me_pxl.png" alt="d" class="img-responsive">
        </div>
    </div>
  </div>
</div>


<style>
  .about-wrapper {
    display: flex;
    flex-wrap: wrap;
    align-items: flex-start;
    gap: 2rem;
  }
  .about-content {
    flex: 1 1 300px;
  }
  .about-image-container {
    flex: 0 0 300px;
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  .about-image {
    width: 300px;
    height: 300px;
    border-radius: 50%;
    overflow: hidden;
  }
  .about-image img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
  .image-caption {
    margin-top: 15px;
    font-style: italic;
    text-align: center;
    font-size: 0.9em;
    max-width: 300px;
  }
  @media screen and (max-width: 768px) {
    .about-wrapper {
      flex-direction: column-reverse;
    }
    .about-image-container {
      margin-bottom: 1rem;
    }
  }
</style>
