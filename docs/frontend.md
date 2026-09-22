# Frontend — NeuralArtX

This document covers the responsive UI: section layout, Swiper carousels, form validation, and the JavaScript that ties the frontend to the backend API.

---

## Section Structure

The site is a single-page layout with five scrollable sections, a fixed navigation bar, and three utility icons (wishlist, cart, user login):

```
┌─────────────────────────────────────────────────┐
│  Nav: Art.  Home  About  Products  Review  Contact  ♡ 🛒 👤  │
├─────────────────────────────────────────────────┤
│                                                  │
│  [Hero Section]         Home                     │
│  "Intricate & Fine Art"                          │
│  CTA: Shop Now                                   │
│                                                  │
├─────────────────────────────────────────────────┤
│  [About Section]        About Us                 │
│  Video + "Why choose us?"                        │
│                                                  │
├─────────────────────────────────────────────────┤
│  [Products Section]     Latest Products          │
│  3-column grid (desktop) → 1-column (mobile)     │
│  Each card: image, discount badge, price,        │
│  hover → wishlist / add-to-cart / share           │
│                                                  │
├─────────────────────────────────────────────────┤
│  [Review Section]       Customer Reviews         │
│  Star ratings + testimonials (Swiper carousel)   │
│                                                  │
├─────────────────────────────────────────────────┤
│  [Contact Section]      Contact Us               │
│  Validated form (name, email, number, message)   │
│                                                  │
├─────────────────────────────────────────────────┤
│  [Footer]                                        │
│  Quick Links / Extra Links / Locations / Contact │
└─────────────────────────────────────────────────┘
```

Navigation uses anchor links (`#home`, `#about`, etc.) for smooth in-page scrolling. The nav bar gets a sticky `.active` class after 100px of scroll via a `window.onscroll` listener.

## Responsive Carousels (Swiper.js)

Both the product catalog and the review section use Swiper.js with breakpoint-based `slidesPerView`. The same config pattern is used in both, with autoplay on a 4-second interval:

```javascript
var swiper = new Swiper(".product-slider", {
    slidesPerView: 3,
    loop: true,
    spaceBetween: 10,
    autoplay: {
        delay: 4000,
        disableOnInteraction: false
    },
    navigation: {
        nextEl: ".swiper-button-next",
        prevEl: ".swiper-button-prev"
    },
    breakpoints: {
        0:    { slidesPerView: 1 },
        550:  { slidesPerView: 2 },
        800:  { slidesPerView: 3 }
    }
});
```

**Why Swiper over a custom carousel:** Swiper handles touch gestures, momentum scrolling, loop wrapping, and responsive breakpoints out of the box. Writing that from scratch in vanilla JS for a course project would have been a week of work with a worse result.

**Why `disableOnInteraction: false`:** by default, Swiper stops autoplaying once a user swipes manually. Setting this to `false` means the carousel resumes autoplay after the user stops interacting — better for a storefront where you want continuous product visibility.

The CSS defines all theme-dependent colours on `body` and overrides them on `body.active`:

```css
body {
    --bg: #fff;
    --text: #333;
    --card-bg: #f7f7f7;
}

body.active {
    --bg: #1a1a2e;
    --text: #eee;
    --card-bg: #16213e;
}
```

Every element references these variables — `background: var(--bg)`, `color: var(--text)` — so a single class toggle repaints the entire page.

## Product Cards

Each product card is a self-contained `.box` element with three interactive overlays on hover:

```html
<div class="box">
    <span class="discount">-20%</span>
    <div class="image">
        <img src="images/feeling-blue.jpg">
        <div class="icons">
            <a href="#" class="fas fa-heart"></a>
            <a href="#" class="cart-btn">add to cart</a>
            <a href="#" class="fas fa-share"></a>
        </div>
    </div>
    <div class="content">
        <h3>Feeling Blue</h3>
        <div class="price">Rs 10,000/- <span>Rs 12,000/-</span></div>
    </div>
</div>
```

The `.icons` div is positioned absolutely over the image and slides up on `.box:hover` via a CSS transition. The discount badge is positioned top-left with `position: absolute`.

**"Add to Cart" connects to the backend:** clicking the cart button triggers the `addToCart()` function from the [JWT auth flow](jwt-auth.md), which sends a token-authenticated POST to `/api/orders`. If the user isn't logged in (no token), they're redirected to the registration/login page.

## Contact Form Validation

Client-side regex validation runs on every field before the form reaches the backend:

```javascript
function validateForm() {
    var name = document.getElementById("name").value;
    var email = document.getElementById("email").value;
    var number = document.getElementById("number").value;
    var message = document.getElementById("message").value;

    var emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    var phoneRegex = /^[0-9]{10}$/;

    if (name.trim() === '') {
        alert('Please enter your name');
        return false;
    }
    if (!emailRegex.test(email)) {
        alert('Please enter a valid email');
        return false;
    }
    if (!phoneRegex.test(number)) {
        alert('Please enter a valid 10-digit phone number');
        return false;
    }
    if (message.trim() === '') {
        alert('Please enter a message');
        return false;
    }

    // Validation passed — submit to backend
    savedata(name, email, number, message);
    return true;
}
```

**Why client-side and server-side validation both exist:** client-side catches obvious mistakes before a network round-trip (better UX). Server-side validation (parameterised queries) is the actual security layer — client-side JS can be bypassed by anyone with browser dev tools.

## Responsive Design

The layout uses percentage-based widths and media-query breakpoints rather than fixed pixel widths:

- **Desktop (800px+):** 3-column product grid, full navigation bar, side-by-side About section (video + text)
- **Tablet (550–800px):** 2-column product grid, navigation collapses to hamburger
- **Mobile (< 550px):** single-column stacked layout, full-width cards, Swiper shows 1 slide at a time

The navigation bar switches to a toggleable hamburger menu on smaller screens:

```javascript
let navbar = document.querySelector('.navbar');

document.querySelector('#menu-bar').onclick = () => {
    navbar.classList.toggle('active');
};

// Close navbar on scroll (prevents it from covering content)
window.onscroll = () => {
    navbar.classList.remove('active');
};
```

## Screenshots

### Home (Hero Section)
![Home](../screenshots/01-home.png)

### Product Grid with Hover Actions
![Products](../screenshots/03-products-grid.png)
![Cart Interaction](../screenshots/11-cart-hover.jpeg)

### Contact Form
![Contact](../screenshots/06-contact-form.png)
![Contact — Live](../screenshots/12-contact-live.jpeg)

### Footer
![Footer](../screenshots/09-footer.png)

---

← Back to [README](../README.md)
