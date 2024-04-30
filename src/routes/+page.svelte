<script>
    import { onMount } from "svelte";
    import {
        parse,
        HtmlGenerator,
    // @ts-ignore
    } from "https://cdn.jsdelivr.net/npm/latex.js/dist/latex.mjs";
    import Highlight from "svelte-highlight";
    import latex from "svelte-highlight/languages/latex";
    import { PUBLIC_API_URL } from "$env/static/public";

    const API_URL = PUBLIC_API_URL;
    let message = "";
    let result = "";
    let style = "APA Style";
    let loading = false;
    /**
     * @param {string} prompt
     */
    async function* streamChatCompletions(prompt) {
        const url = `${API_URL}/request-message/`;
        var data = { message: prompt, style: style };
        //fetch stream response
        const response = await fetch(url, {
            method: "POST",
            headers: {
                "Content-Type": "application/json",
            },
            body: JSON.stringify(data),
        });

        if(response.body === null) {
            throw new Error("Response body is null");
        }

        const reader = response.body.getReader();
        while (true) {
            const { done, value } = await reader.read();
            if (done) {
                break;
            }
            yield new TextDecoder().decode(value);
        }
    }

    /**
     * @param {string} text
     */
    function cleanLatex(text) {
        let textTmp = text;
        //remove \usepackage{...}
        textTmp = textTmp.replaceAll(/\\usepackage{.*?}/g, "");
        textTmp = textTmp.replaceAll(/\\usepackage[.*?]{.*?}/g, "");

        textTmp = textTmp.replaceAll(/\\geometry{.*?}/g, "");

        //replace & with /& but not in \&
        textTmp = textTmp.replaceAll(" &", " \\&");
        
        //replace number with % with \\%, example 100% -> 100\\%, using regex
        textTmp = textTmp.replaceAll(/(\d+)%/g, "$1\\%");

        textTmp = textTmp.replaceAll("_", " \\_");
        

        textTmp = textTmp.replaceAll(" [", " {[");
        textTmp = textTmp.replaceAll("] ", "]} ");

        //change \begin{thebibliography} and \end{thebibliography} to \begin{itemize} and \end{itemize}
        if (textTmp.includes("\\begin{thebibliography}")) {
            //remove {...} after \begin{thebibliography}
            textTmp = textTmp.replace(/\\begin{thebibliography}{.*?}/g, "");
            textTmp = textTmp.replaceAll(
                "\\begin{thebibliography}",
                "\\begin{itemize}",
            );
            textTmp = textTmp.replaceAll(
                "\\end{thebibliography}",
                "\\end{itemize}",
            );
            //replace \bibitem with \item
            textTmp = textTmp.replaceAll("\\bibitem", "\\item");
        }

        //if textTmp contains \url{...}
        if (textTmp.includes("\\url{")) {
            //remove \url{...} and replace with ...
            textTmp = "\\usepackage{hyperref}\n"+textTmp.replaceAll(/\\url{.*?}/g, "...");
        }

        if (textTmp.includes("\\begin{picture}")) {
            //remove \url{...} and replace with ...
            textTmp = "\\usepackage{calc,pict2e,picture}\n"+textTmp.replaceAll(/\\url{.*?}/g, "...");
        }

        return textTmp;
    }

    /**
     * @param {string} text
     */
    function refactorLatex(text) {
        let textTmp = text;
        //count bracket
        let countOpen = (textTmp.match(/{/g) || []).length;
        let countClose = (textTmp.match(/}/g) || []).length;
        if (countOpen > countClose) {
            textTmp += "}\n";
        }

        //count word /begin{...} and add \end{...} with same name
        let begin = textTmp.match(/\\begin{.*?}/g) || [];
        let end = textTmp.match(/\\end{.*?}/g) || [];
        for (let i = 0; i < begin.length; i++) {
            let name = begin[i].replace("\\begin{", "").replace("}", "");
            // @ts-ignore
            if (!end.includes(`\\end{${name}}`)) {
                textTmp += `\\end{${name}}\n`;
            }
        }

        // let countBegin = (textTmp.match(/\\begin/g) || []).length;
        // let countEnd = (textTmp.match(/\\end/g) || []).length;
        // if (countBegin > countEnd) {
        //     textTmp += "\n \\end{document}";
        // }
        return textTmp;
    }

    /**
     * @param {string} text
     */
    function renderLatex(text) {
        let generator = new HtmlGenerator({ hyphenate: false });
        let doc = parse(text, {
            generator: generator,
        }).htmlDocument();
        doc.head.appendChild(
            generator.stylesAndScripts(
                "https://cdn.jsdelivr.net/npm/latex.js@0.12.4/dist/",
            ),
        );
        /**
         * @type {HTMLIFrameElement}
         */
        // @ts-ignore
        const resultEl = document.getElementById("result") || {};
        if (resultEl) {
            resultEl.srcdoc = doc.documentElement.outerHTML;
        }
    }

    async function getResponse() {
        if (loading) {
            return;
        }
        result = "";

        if (message === "") {
            const randomTheme = [
                "teknologi",
                "AI",
                "kesehatan",
                "pertanian",
                "perkebunan",
                "energi",
                "sosial",
                "kemiskinan",
                "pendidikan",
                "batubara",
                "lingkungan",
                "kendaraan listrik",
                "smartphone",
                "IoT",
                "anime",
                "olahraga",
                "penipuan",
            ];
            message = `topik ${randomTheme[Math.floor(Math.random() * randomTheme.length)]}`;
        }
        
        loading = true;
        try {
            const stream = streamChatCompletions(message);
            for await (const chunk of stream) {
                const ck = chunk.replaceAll("```", "").replaceAll(":FINISH:", "");
                result += ck;
                result = cleanLatex(result);
                try {
                    const refactorResult = refactorLatex(result);
                    renderLatex(refactorResult);
                } catch (error) {
                    console.log(error);
                    
                }
            }
            try {
                result = cleanLatex(refactorLatex(result));
                renderLatex(result);
            } catch (error) {
                console.log(error);
                alert(error);
            }
        } catch (error) {
            console.log(error);
            alert(error);
        }
        
        loading = false;
    }

    onMount(async () => {});
</script>

<svelte:head>
    <title>Buat karya ilmiah abal-abal</title>
    <link
        rel="stylesheet"
        href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/default.min.css"
    />
</svelte:head>
<div>
    <div class="my-2 px-6 py-4">
        <h1 class="text-2xl font-bold text-center">
            Buat artikel ilmiah abal-abal 🤖✍️
        </h1>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-2">
            <div class="w-full py-2">
                <div class="my-2">
                    <label
                        for="countries"
                        class="block mb-2 text-sm font-medium text-gray-900 dark:text-white"
                        >Format Penulisan</label
                    >
                    <select
                        bind:value={style}
                        id="countries"
                        class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                    >
                        <option value="APA Style">APA Style</option>
                        <option value="ASA Style">ASA Style</option>
                        <option value="Vancouver Style">Vancouver Style</option>
                        <option value="Harvard Style">Harvard Style</option>
                        <option value="Chicago Style">Chicago Style</option>
                    </select>
                    <div class="my-2">
                        <label
                            for="message"
                            class="block mb-2 text-sm font-medium text-gray-900 dark:text-white"
                            >Tema Tulisan</label
                        >
                        <input
                            type="text"
                            maxlength="200"
                            id="message"
                            bind:value={message}
                            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
                            placeholder="Topik Teknologi, Kesehatan, Pendidikan, dll"
                            required
                        />
                    </div>
                    <button
                        disabled={loading}
                        on:click={getResponse}
                        class="text-white my-2 bg-gradient-to-r from-blue-500 to-teal-500 hover:bg-gradient-to-l focus:ring-4 focus:outline-none focus:ring-blue-200 dark:focus:ring-blue-800 font-medium rounded-lg text-sm px-5 py-2.5 text-center me-2 mb-2"
                    >
                        {#if loading}
                            <span>Loading...</span>
                        {:else}
                            Generate 📝
                        {/if}
                    </button>
                </div>
                <div class="w-full overflow-auto" style="height: 58vh;">
                    <!-- <pre class="language-latex">
                        <code>
                            {result}
                        </code>
                    </pre> -->
                    <Highlight language={latex} code={result} />
                </div>
            </div>
            <div class="w-full overflow-auto py-2 ">
                <label
                    for="countries"
                    class="block my-2 text-sm font-medium text-gray-900 dark:text-white"
                    >Result</label
                >
                <div class="relative" style="height: 78vh;">
                    <iframe
                    class="w-full"
                    style="height: 100%;"
                    id="result"
                    sandbox="allow-same-origin allow-scripts"
                    frameborder="0"
                ></iframe>
                    
                </div>
                
                
            </div>
        </div>
        <p class="text-red-500 font-bold">
            * Semua tulisan/artikel yang dihasilkan di website ini adalah hasil AI, tentunya kualitasnya tidak layak bahkan bisa dibilang "sampah", jadi jangan gunakan hasil dari website ini di lingkungan akademis.
        </p>
        <p class="text-red-500 font-bold">
            * Website ini merupakan proof of concept dari penggunaan dan implementasi AI dalam lingkup content generation.
        </p>
        <p class="text-red-500 font-bold">
            * Pengembang wesbite tidak bertanggung jawab atas hasil yang dihasilkan oleh AI/Website ini.
        </p>
    </div>
</div>
