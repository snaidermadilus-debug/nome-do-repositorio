nano wiki_bot.py


import wikipedia
import os

wikipedia.set_lang("pt")  # Define a língua para português

def pesquisar_wikipedia():
    while True:
        pergunta = input("Pergunte algo (ou digite 'sair' para encerrar): ").strip()
        
        if pergunta.lower() == "sair":
            print("Encerrando o programa. Até mais!")
            os.system('termux-tts-speak "Encerrando o programa. Até mais!"')
            break

        try:
            # Resumo curto (3 frases)
            resumo_curto = wikipedia.summary(pergunta, sentences=3)
            print("\nResposta do Wikipédia (curta):")
            print(resumo_curto)
            os.system(f'termux-tts-speak "{resumo_curto}"')

            # Pergunta se quer a versão longa
            opcao = input("\nQuer ouvir a versão longa? (sim/não) ").strip().lower()
            if opcao == "sim":
                resumo_longo = wikipedia.summary(pergunta, sentences=10)  # Mais detalhado
                print("\nVersão longa:")
                print(resumo_longo)
                os.system(f'termux-tts-speak "{resumo_longo}"')
            else:
                print("Continuando para a próxima pergunta...\n")

        except wikipedia.exceptions.DisambiguationError as e:
            print("\nSua pergunta pode se referir a várias coisas. Seja mais específico!")
            print("Sugestões:", e.options[:5])
            os.system('termux-tts-speak "Sua pergunta pode se referir a várias coisas. Seja mais específico."')

        except wikipedia.exceptions.PageError:
            print("\nNão encontrei nada no Wikipédia sobre isso.")
            os.system('termux-tts-speak "Não encontrei nada no Wikipédia sobre isso."')

pesquisar_wikipedia()



WikiBot Termux

Um bot simples em Python que busca respostas no Wikipédia em português e fala as respostas usando o Termux:API. Ele mostra um resumo curto da resposta e dá a opção de ouvir uma versão longa.

Funcionalidades

Busca informações diretamente no Wikipédia em português.

Mostra um resumo curto (3 frases).

Pergunta se você quer ouvir a versão longa (10 frases).

Fala a resposta usando o Termux TTS.

Permite digitar SAIR para encerrar o programa.


Requisitos

Para rodar este programa, você precisa ter:

Python 3 instalado no Termux ou PC.

Termux:API instalada no Termux:

pkg install termux-api

Biblioteca wikipedia do Python:

pip install wikipedia


Como usar

2. use isso no termux aí você coloca o código lá em cima👆 que eu coloquei

👉1 nano wiki_bot.py


3. Execute o programa:

python wiki_bot.py


4. Digite qualquer pergunta. Exemplo:

Pergunte algo (ou SAIR):


5. O bot vai mostrar o resumo curto e falar a resposta.


6. Ele perguntará se você quer ouvir a versão longa. Digite sim ou não.


7. Para sair, digite SAIR.



Observações

Funciona melhor no Termux por causa da integração com o TTS (termux-tts-speak).

Caso a pergunta seja ambígua, ele sugerirá algumas opções.

Se não encontrar resultados, ele avisará.
