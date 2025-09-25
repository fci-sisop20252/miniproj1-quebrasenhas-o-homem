# Relatório: Mini-Projeto 1 - Quebra-Senhas Paralelo

**Aluno(s):** Vinicius Kenji Anzai Machado (10443471)
---

## 1. Estratégia de Paralelização


**Como você dividiu o espaço de busca entre os workers?**

 Esse espaço foi dividido de forma que peguei o numero total de combinações possiveis e dividi pelo numero de workers (caso tenha resto os primeiros workers recebem mais senhas). Desta forma cada worker recebe um intervalo de senhas por meio de um index inicial e outro final.

**Código relevante:** Cole aqui a parte do coordinator.c onde você calcula a divisão:
```c
long long passwords_per_worker = total_space / num_workers;
long long remaining = total_space % num_workers;
long long startIndex = 0;

for (int i = 0; i < num_workers; i++) {

    long long numPasswords = passwords_per_worker + (i < remaining ? 1 : 0);
    long long endIndex = startIndex + numPasswords - 1;

    char startPassword[128], endPassword[128];
    index_to_password(startIndex, charset, charset_len, password_len, startPassword);
    index_to_password(endIndex,   charset, charset_len, password_len, endPassword);
    printf("Worker %d: intervalo [%s ... %s] (índices %lld ... %lld)\n",i, startPassword, endPassword, startIndex,endIndex);

    startIndex = endIndex + 1;

}
```

---

## 2. Implementação das System Calls

**Descreva como você usou fork(), execl() e wait() no coordinator:**

Utilizei o _fork()_ dentro de um loop para que cada chamada criasse um processo fiho, o qual representa um worker responsavel por um espaço de busca. O _excel()_ foi utilizado para que a execução fosse trocada pelo programa worker, os parametros foram: hash, senha inicial, final do intervalo, charset, tamanho da senha, e id do worker. No processo pai guardei todos os PID dos workers e utilizei _wait()_ em um laço para certificar que todos os processos filhos (workers) terminassem

**Código do fork/exec:**
```c
long long startIndex = 0;

for (int i = 0; i < num_workers; i++) {
    long long numPasswords = passwords_per_worker + (i < remaining ? 1 : 0);
    long long endIndex = startIndex + numPasswords - 1;

    char startPassword[128], endPassword[128];
    index_to_password(startIndex, charset, charset_len, password_len, startPassword);
    index_to_password(endIndex,   charset, charset_len, password_len, endPassword);

    char id[16], size_str[16];
    sprintf(id, "%d", i);
    sprintf(size_str, "%d", password_len);

    pid_t pid = fork();
    if (pid < 0) {
        perror("Erro ao criar fork");
        exit(1);
    } else if (pid == 0) {
        execl("./worker", "worker",
              target_hash, startPassword, endPassword,
              charset, size_str, id, (char *)NULL);
        perror("Erro no execl");
        _exit(1);
    } else {
        workers[i] = pid;
        printf("Worker %d (PID=%d) intervalo [%s ... %s]\n",
               i, pid, startPassword, endPassword);
    }

    startIndex = endIndex + 1;
}
```

---

## 3. Comunicação Entre Processos

**Como você garantiu que apenas um worker escrevesse o resultado?**

 Eu utilizei as flags O_CREAT | O_EXCL | O_WRONLY na abertura do arquivo password_found.txt, assim, o arquivo só é criado caso ele ainda não exista, se ele existir ocorre erro. Ou seja, em um cenário que vário workers estão funcionando ao mesmo tempo e mais de um achou a senha em quase o mesmo tempo, apenas o primeiro worker irá conseguir criar o arquivo, assim, impedindo uma condição de corrida onde dois processos tentam escrever no mesmo arquivo ao mesmo tempo.

**Como o coordinator consegue ler o resultado?**

 O coordinator consegue ler o resultado abrindo e lendo o arquivo password_fount.txt após o final da operação de todos workers. Ele encontra dentro do arquivo o conteudo desta forma: idDoworker:senha, assim consegue separar com : e possuir tanto a informação da senha quanto quem encontrou ela. o coordinator também transforma em hash MD5 novamente para comparar se está certo o resultado.

---

## 4. Análise de Performance
Complete a tabela com tempos reais de execução:
O speedup é o tempo do teste com 1 worker dividido pelo tempo com 4 workers.

| Teste | 1 Worker | 2 Workers | 4 Workers | Speedup (4w) |
|-------|----------|-----------|-----------|--------------|
| Hash: 202cb962ac59075b964b07152d234b70<br>Charset: "0123456789"<br>Tamanho: 3<br>Senha: "123" | 0.000000s | 0.000000s | 0.000000s | 0 |
| Hash: 5d41402abc4b2a76b9719d911017c592<br>Charset: "abcdefghijklmnopqrstuvwxyz"<br>Tamanho: 5<br>Senha: "hello" | 5s | 8s | 2s | 2.5 |

**O speedup foi linear? Por quê?**

Não, o speedup não foi linear, isso pode ser observado com o fato de que mesmo aumentando o numero de workers o ganho de desempenho não foi proporcional. Isso ocorre pois para executar funções como fork e execl ou utilizar um arquivo é necessário uma troca de contexto, consumindo muito tempo, este processo é conhecido como overhead. Outra questão é que nem sempre o espaço de busca pode ser dividido igualmente pelos workers, então alguns trabalham mais que outros. Dessa forma, concluimos que o speedup não é linear por conta de gastos derivados das funções essenciais para o paralelismo

---

## 5. Desafios e Aprendizados
**Qual foi o maior desafio técnico que você enfrentou?**
 Meu maior desafio técnico foi compreender e aplicar as funções fork() e execl(), já que eu tinha dificuldade em visualizar o que acontecia em cada processo, por conta do paralelismo e do coordinator e workers funcioando simultaneamente. Para solucionar minhas dificuldades eu apliquei mensagens de depuração e esudei melhor tanto as documentações fornecidas quanto fontes exteriores, assim, ganhei uma visão melhor do paralelismo. 

---

## Comandos de Teste Utilizados

```bash
# Teste básico
./coordinator "900150983cd24fb0d6963f7d28e17f72" 3 "abc" 2

# Teste de performance
time ./coordinator "202cb962ac59075b964b07152d234b70" 3 "0123456789" 1
time ./coordinator "202cb962ac59075b964b07152d234b70" 3 "0123456789" 4

# Teste com senha maior
time ./coordinator "5d41402abc4b2a76b9719d911017c592" 5 "abcdefghijklmnopqrstuvwxyz" 4
```
---

**Checklist de Entrega:**
- [X] Código compila sem erros
- [X] Todos os TODOs foram implementados
- [X] Testes passam no `./tests/simple_test.sh`
- [X] Relatório preenchido