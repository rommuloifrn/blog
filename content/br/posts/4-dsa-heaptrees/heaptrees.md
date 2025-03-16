+++
date = '2025-03-16T02:11:42-03:00'
draft = true
title = 'Árvores heap'
+++

# Árvores heap (heap trees)

## Introdução

A árvore heap é uma forma de implementar o conceito de **fila de prioridade**. Note que na fila do banco existem duas prioridades: Especial (idosos, gestantes...) e comum. Já a nossa heap pode ter vários níveis.

--- imagem

## Representação de prioridade

A prioridade de um nó na árvore heap é representada por um número: Se a prioridade são os números maiores, chamamos de **min-heap (heap mínimo, menores primeiro)**. Já se a prioridade são os números menores, chamamos de **max-heap (heap máximo)**.

Aqui irei considerar somente a min-heap, pois é o que Robinson [^1] usa.

[^1]: Um professor aí.

## Inserção

## Balanceamento

Para uma heap ser considerada *balanceada* [^2], há duas propriedades que ela precisa satisfazer:

- Heap-order: Todo nó deve estar acima dos de menor prioridade
- Completude: a árvore deve ser preenchida da esquerda para a direita

[^2]: Balanceada é uma árvore que não está degenerada, que satisfaz sua forma perfeita.

## Remoção