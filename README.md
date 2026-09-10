# Relatório: Processamento Distribuído de Logs com MPI

## Introdução

Nesta atividade o objetivo foi criar um programa em Python usando a biblioteca mpi4py para simular o processamento de um grande volume de logs de forma distribuída. A ideia central era dividir um conjunto de logs entre vários processos usando a operação coletiva Scatter, fazer cada processo contar quantos erros (status 404 ou 500) apareceram na sua parte e depois cada um informar o resultado no terminal.

## Desenvolvimento

O ponto de partida foi o template fornecido, que já tinha a estrutura básica do programa com os comentários indicando onde cada parte da lógica deveria entrar. A partir dele fui completando as etapas pedidas.

Primeiro implementei a função gerar_logs, que monta uma lista de strings no formato IP METODO ENDPOINT STATUS. Para isso usei random.choice para sortear aleatoriamente um IP da lista, um método (GET ou POST), um endpoint e um status. A lista de status foi montada com mais ocorrências de 200 do que de 404 e 500, seguindo o padrão que já estava no template, para simular um cenário onde a maioria das requisições tem sucesso.

Depois veio a parte da divisão dos dados. Como só o processo de rank 0 gera o dataset, é ele quem precisa dividir a lista de logs em partes iguais, uma para cada processo que vai participar da execução. Fiz isso calculando o tamanho de cada fatia como o total de logs dividido pelo número de processos (size) e depois montando uma lista de listas, uma fatia para cada rank. Essa lista de fatias é justamente o parâmetro que o Scatter espera receber, já que ele distribui um elemento da lista para cada processo, na ordem dos ranks.

A distribuição em si ficou bem direta, bastando chamar comm.scatter passando a lista dividida e definindo root=0, já que é o processo 0 quem tem os dados completos antes da distribuição. Depois dessa chamada, cada processo, incluindo o próprio processo 0, fica com a variável logs_locais contendo apenas a sua fatia.

Para o processamento local, cada processo percorre sua lista de logs, separa os campos da string usando split, pega o quarto campo que é o status, e verifica se ele é igual a 404 ou 500. Se for, incrementa um contador de erros. No final desse laço cada processo já sabe quantas linhas recebeu (usando len de logs_locais) e quantos erros encontrou.

Por fim, a impressão do resultado já estava praticamente pronta no template, só precisando das variáveis logs_locais e erros calculadas nas etapas anteriores. Como cada processo imprime seu próprio resultado diretamente, sem precisar mandar mensagem separada para o master nem usar Gather ou Reduce, o programa ficou de acordo com o que foi pedido no enunciado, que deixa claro que não é necessário consolidar o resultado em um único processo.

## Testes realizados

Rodei o programa localmente com o comando mpirun usando diferentes quantidades de processos para validar o comportamento. Com 4 processos e TOTAL_LOGS igual a 100000, cada processo recebeu exatamente 25000 linhas, como esperado pela divisão igualitária. Testei também com 5 processos e cada um recebeu 20000 linhas, confirmando que a divisão está funcionando corretamente para diferentes valores de size.

Um detalhe que percebi durante os testes é que como cada processo imprime seu resultado de forma independente e simultânea, às vezes as linhas de saída de processos diferentes aparecem meio misturadas no terminal, sem quebra de linha entre elas. Isso é esperado em programas paralelos que escrevem na saída padrão ao mesmo tempo e não chega a ser um problema, já que o enunciado pede que cada processo imprima seu próprio resultado, sem exigir uma ordem específica.

## Dificuldades encontradas

A parte que exigiu mais atenção foi entender a diferença entre gerar os dados soltos no processo 0 e como transformar isso em algo que o Scatter conseguisse distribuir corretamente. No começo cheguei a pensar em enviar a lista inteira para todo mundo e cada processo filtrar sua parte, mas isso não seria usar o Scatter da forma correta, já que o objetivo dele é justamente enviar pedaços diferentes para cada processo em uma única chamada coletiva. Depois que entendi que o Scatter espera uma lista com exatamente size elementos, sendo um para cada rank, o resto ficou mais tranquilo de implementar.

## Conclusão

A atividade ajudou bastante a fixar o conceito de particionamento e distribuição de dados usando MPI. Ficou claro como o Scatter simplifica bastante o processo de dividir um trabalho entre vários processos, evitando que o desenvolvedor precise implementar manualmente o envio de mensagens ponto a ponto para cada worker. Também ficou evidente a vantagem de processar os dados em paralelo, já que cada processo trabalha apenas com sua fatia, o que é essencial quando se trabalha com volumes de dados muito grandes, como o próprio enunciado menciona no contexto de sistemas reais.
