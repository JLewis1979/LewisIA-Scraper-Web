// main.js
const Apify = require('apify');
const { log } = Apify.utils;

Apify.main(async () => {
    log.info('🚀 Actor iniciado...');

    // 1. OBTENER INPUT: La URL de la página de listado
    const input = await Apify.getInput();
    const startUrl = input.startUrl;

    if (!startUrl) {
        log.error('❌ ERROR: "startUrl" no fue proveída en el input. Terminando actor.');
        await Apify.setValue('OUTPUT', { error: 'startUrl is required' });
        return;
    }
    log.info(`🔗 URL de inicio recibida: ${startUrl}`);

    // 2. INICIALIZAR REQUEST QUEUE
    const requestQueue = await Apify.openRequestQueue();
    await requestQueue.addRequest({
        url: startUrl,
        userData: { label: 'LIST_PAGE' },
    });

    // 3. CONFIGURAR PUPPETEERCRAWLER
    const crawler = new Apify.PuppeteerCrawler({
        requestQueue,
        launchContext: {
     
            // headless: 'new',
            args: ['--no-sandbox', '--disable-setuid-sandbox'],
        },
        minConcurrency: 1,
        maxConcurrency: 3,
        maxRequestRetries: 3,
        handlePageTimeoutSecs: 120,

        // 4. FUNCIÓN PARA MANEJAR CADA PÁGINA
        handlePageFunction: async ({ request, page }) => {
            const { url, userData: { label } } = request;
            log.info(`📄 Procesando [${label}]: ${url}`);

            if (label === 'LIST_PAGE') {
                // --- LÓGICA PARA LA PÁGINA DE LISTADO ---
                log.info('   Es una página de listado. Buscando enlaces a artículos...');

                // ***** ¡¡¡DEBES REEMPLAZAR ESTE SELECTOR CSS!!! *****
                const ARTICLE_LINK_SELECTOR_ON_LIST_PAGE = 'a.mnt¡Excelente! Vamosl-card-list-items[href^="/"]';

                const articleUrls = await page.evaluate((selector) => {
                    const links = [];
                    document.querySelectorAll(selector).forEach(anchor => {
                        if (anchor.href) links.push(anchor.href);
                    });
                    return links;
                }, ARTICLE_LINK_SELECTOR_ON_LIST_PAGE);

                log.info(`   Se encontraron ${articleUrls.length} enlaces de artículos.`);
                for (const articleUrl of articleUrls) {
                    await requestQueue.addRequest({
                        url: articleUrl,
                        userData: { label: 'DETAIL_PAGE' },
                    });
                }
            } else if (label === 'DETAIL_PAGE') {
                // --- LÓGICA PARA LA PÁGINA DE DETALLE DEL ARTÍCULO ---
                log.info('   Es una página de detalle. Extrayendo datos...');

                // ***** ¡¡¡DEBES REEMPLAZAR ESTOS SELECTORES CSS!!! *****
                const TITLE_SELECTOR = 'h1.article-title';             
                const CONTENT_SELECTOR = 'div.article-content-body';   
                const DATE_SELECTOR = 'span.publish-date';             
                const AUTHOR_SELECTOR = 'span.author-name';            

                const extractedData = await page.evaluate((titleSel, contentSel, dateSel, authorSel) => {
                    const title = document.querySelector(titleSel)?.innerText.trim();
                    
                    // Para el contenido, podemos intentar ser un poco más selectivos
                    let contentText = '';
                    const contentElement = document.querySelector(contentSel);
                    if (contentElement) {
                        // Extraer texto de párrafos y elementos de lista dentro del contenedor de contenido
                        contentElement.querySelectorAll('p, li, h1, h2, h3, h4, h5, h6').forEach(el => {
                            contentText += el.innerText.trim() + '\n\n';
                        });
                        if (!contentText) { // Fallback si no hay p, li, etc.
                            contentText = contentElement.innerText.trim();
                        }
                    }

                    const publicationDate = document.querySelector(dateSel)?.innerText.trim();
                    const author = document.querySelector(authorSel)?.innerText.trim();

                    return { title, contentText, publicationDate, author };
                }, TITLE_SELECTOR, CONTENT_SELECTOR, DATE_SELECTOR, AUTHOR_SELECTOR);

                // Guardar los datos en el Dataset
                await Apify.pushData({
                    scrapedUrl: url,
                    title: extractedData.title,
                    contentText: extractedData.contentText?.trim(),
                    publicationDate: extractedData.publicationDate,
                    author: extractedData.author,
                });
                log.info(`   👍 Datos guardados para: ${extractedData.title || url}`);
            }
        },

        // Función para manejar errores durante el procesamiento de una página
        handleFailedRequestFunction: async ({ request, error }) => {
            log.error(`❌ Falló la solicitud para ${request.url}: ${error.message}`);
        },
    });

    // 5. INICIAR EL CRAWLER
    log.info('🏁 Iniciando el crawler...');
    await crawler.run();
    log.info('🎉 Crawler finalizado.');
});
