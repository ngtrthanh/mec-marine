# MEC Marine

Vietnamese maritime equipment and services website with a 3D bridge, antenna deck, and safety-equipment area. All 113 catalogue entries are accessible through 37 illustrated stations and model selectors.

**Deploy the ready-built `dist/` directory.** For Cloudflare Pages with GitHub, use framework None, build command `exit 0`, output `dist`, and `SKIP_DEPENDENCY_INSTALL=1`.

See [the complete GitHub + Cloudflare guide](docs/CLOUDFLARE-GITHUB.md).

## Local development

Install Python requirements and run `npm ci`. Use `npm run dev` to preview. Use `npm run build:content` to regenerate the static output and optimized 3D bundles.

## Content and limitations

- Manufacturer product images and catalogue text originate from MEC's website.
- Newly generated service photographs are illustrative, not evidence of actual MEC projects.
- 3D locations and shapes are illustrative, not installation drawings.
- Forms prepare email drafts; no email delivery backend, CRM, or conversion analytics is configured.
- Three.js licensing is retained under `dist/assets/vendor/THREE-LICENSE.txt`.
