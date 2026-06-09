# Estratégia de ML para o IAnamnesis

Faz muito sentido! Esse é um padrão bem sólido — **sistema determinístico como núcleo, ML como camada de suporte incremental**. É inclusive o approach mais seguro para software crítico.

---

## O que você está descrevendo tem nome

**Human-in-the-Loop (HITL)** com **Expert Feedback** — onde:

- O sistema programático resolve com garantia
- O ML atua nos **casos ambíguos ou de fronteira**
- Especialistas validam as predições do ML ao longo do tempo

---

## Arquitetura recomendada

```
[Input]
    ↓
[Sistema determinístico] ──→ resultado garantido (maioria dos casos)
    ↓
[Casos ambíguos / baixa confiança]
    ↓
[Modelo ML] ──→ sugere resultado
    ↓
[Especialista valida] ──→ gera label de qualidade
    ↓
[Dataset cresce] ──→ modelo melhora
    ↓
[ML assume mais casos com o tempo]
```

---

## Estratégia de coleta de feedback dos especialistas

Como o seu label vem de **especialistas** (dado valioso e escasso), você precisa ser eficiente:

### Active Learning é perfeito aqui
O modelo só pede validação humana nos casos onde ele tem **menos certeza** — economiza o tempo do especialista e maximiza o aprendizado por label gerado.

**Threshold de confiança sugerido:**
```
confiança > 90%  → ML decide sozinho (após maturidade)
confiança 60–90% → ML sugere, especialista confirma rapidamente
confiança < 60%  → sistema determinístico assume
```

### Tipos de feedback para especialistas (do mais simples ao mais rico)

| Tipo | Esforço | Qualidade do label |
|---|---|---|
| Confirmar/rejeitar sugestão do ML | Muito baixo | Média |
| Escolher entre 2–3 opções | Baixo | Boa |
| Corrigir output errado | Médio | Alta |
| Anotar do zero | Alto | Máxima |

> Comece com **confirmar/rejeitar** — escala bem e já gera um dataset útil.

---

## Como treinar sem dados iniciais

Já que você consegue o resultado ótimo programaticamente, você tem algo valioso:

**Você pode gerar seu próprio dataset sintético**

```python
# Lógica determinística vira ground truth
for caso in casos_conhecidos:
    resultado_otimo = sistema_deterministico(caso)
    dataset.append((caso, resultado_otimo))  # label gratuito!
```

Isso te dá um **cold start** sem precisar de nenhum especialista no início. O ML aprende o comportamento do sistema determinístico — e depois os especialistas refinam os casos de borda.

---

## Resumo da estratégia

1. **Agora** → Rodar o sistema determinístico e **logar tudo** (inputs, outputs, contexto)
2. **Cold start** → Usar o próprio sistema determinístico como gerador de labels sintéticos
3. **MVP** → ML treinado nos dados sintéticos entra nos casos ambíguos
4. **Feedback loop** → Especialistas validam via Active Learning, dataset melhora
5. **Maturidade** → ML assume progressivamente mais casos com confiança alta

Qual é a natureza do input do seu sistema? Dados tabulares, texto, sequências temporais? Isso define bastante qual família de modelos faz mais sentido.