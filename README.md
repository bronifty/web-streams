# web-streams

- inspiration [vercel ssr streams](https://vercel.com/docs/concepts/functions/edge-functions/streaming)
- education [web.dev/streams](https://web.dev/streams/)

### Resource Types

- Reading
- Writing
- Transforming

## Text Resource

- example 1 create a readable stream and read it, pushing the output to the DOM

```html
<html>
  <body>
    <div id="output"></div>
    <script>
      const readableStream = new ReadableStream({
        start(controller) {
          controller.enqueue(new TextEncoder().encode("Hello "));
          setTimeout(() => {
            controller.enqueue(new TextEncoder().encode("world"));
            controller.close();
          }, 1000);
        },
      });

      const reader = readableStream.getReader();

      const push = () => {
        reader.read().then(({ done, value }) => {
          if (done) {
            return;
          }
          document.querySelector("#output").textContent +=
            new TextDecoder().decode(value);
          console.log(new TextDecoder().decode(value));
          push();
        });
      };
      push();
    </script>
    <style>
      body {
        background: #222;
        color: #ddd;
        font-size: 2rem;
      }
    </style>
  </body>
</html>
```
