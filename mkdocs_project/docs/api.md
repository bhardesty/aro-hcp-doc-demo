
# API reference

<div class="swagger-fullwidth">
  <div id="swagger-ui"></div>
</div>

<!-- Load Swagger UI CSS & JS from CDN -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swagger-ui-dist/swagger-ui.css">
<script src="https://cdn.jsdelivr.net/npm/swagger-ui-dist/swagger-ui-bundle.js"></script>
<script src="https://cdn.jsdelivr.net/npm/swagger-ui-dist/swagger-ui-standalone-preset.js"></script>

<!-- Initialize Swagger UI -->
<script>
window.onload = function() {
  SwaggerUIBundle({
    url: 'https://raw.githubusercontent.com/Azure/ARO-HCP/refs/heads/main/api/redhatopenshift/resource-manager/Microsoft.RedHatOpenShift/hcpclusters/preview/2024-06-10-preview/openapi.json',
    dom_id: '#swagger-ui',
    presets: [
      SwaggerUIBundle.presets.apis,
      SwaggerUIBundle.SwaggerUIStandalonePreset
    ],
    layout: "BaseLayout"
  });
};
</script>
