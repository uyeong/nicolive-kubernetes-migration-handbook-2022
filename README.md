# 니코니코 생방송 웹 프런트엔드의 쿠버네티스 마이그레이션 핸드북 2022

> 이 문서는 일본어 기술 글 “[ニコニコ生放送 WebフロントエンドのKubernetes移行ハンドブック 2022](https://dwango.github.io/nicolive-kubernetes-migration-handbook-2022/)“을 번역한 글입니다. 번역을 신뢰하지 마세요. 개인적으로 살펴보고 활용하기 위해 번역한 것으로 애매하거나 이해가 확실하지 않은 부분을 가볍게 넘어간 구간이 많습니다.

* [PDF](https://github.com/dwango/nicolive-kubernetes-migration-handbook-2022/raw/main/tex-workspace/article.pdf)

## Develop

```bash
hugo server -D
```

## Deploy

After merging into the main branch, create a release.

## Develop

```bash
pnpm install
```

Generate PDF

* `pnpm run build`
  1. Generate Source Markdown for pandoc
  2. Generate TeX File by pandoc
  3. Update TeX File
  4. Copy PNG / PDF
  5. Generate DVI File (width toc file)
  6. Generate DVI File
  7. Covert DVI to PDF

**Generate Graphic PDF**

1. Download PDF from [diagrams (draw.io)](https://app.diagrams.net/)
2. Separate PDF by `pdfseparate` (poppler)
   * [Docker Image](https://hub.docker.com/r/minidocks/poppler)
3. Crop PDF by `pdf-crop-margins`
   * [Docker Image](https://github.com/Himenon/pdfCropMargins-docker/pkgs/container/pdf-crop-margins)

## LICENSE

Shield: [![CC BY 4.0][cc-by-shield]][cc-by]

This work is licensed under a
[Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
