# toogle-master-monolith
atividade pós tech fiap - fase 1

Analise o código e identifique por que ele é considerado um "monolito". Discuta as vantagens e desvantagens dessa abordagem para um MVP.

**R:** Um monolito é uma aplicação em que todas as responsabilidades são empacotadas, implantadas e escaladas como uma única unidade. O ToggleMaster se enquadra nisso pelos seguintes motivos:

1. Uma única unidade de deploy. Todo o comportamento da aplicação está em um único arquivo, o `app.py`: as rotas HTTP (`/health` e `/flags`), a validação das entradas, o acesso ao banco de dados e a inicialização do schema (`init_db`). Qualquer alteração, por menor que seja, exige um novo deploy da aplicação inteira.
2. Um único processo lógico e um único banco de dados. Todas as funcionalidades compartilham o mesmo runtime e o mesmo schema (a tabela `flags`). Elas são escaladas juntas, e não é possível, por exemplo, dar mais recursos apenas à leitura de flags.
3. Um único repositório, runtime e ciclo de release. Todos os desenvolvedores trabalham na mesma base de código, na mesma linguagem (Python/Flask) e na mesma cadência de entrega. Uma funcionalidade nova depende do mesmo pipeline e do mesmo calendário que as demais.

## Vantagens para um MVP

- **Velocidade:** pouco código e fácil primeiro deploy permitem entregar a primeira versão rapidamente.
- **Operacional:** um processo e um banco são fáceis de rodar localmente e de implantar.
- **Depuração direta:** não há chamadas de rede entre serviços, então não há dificuldade para rastrear erros entre microserviços.
- **Custo baixo:** pequena equipe necessária, menos infraestrutura para provisionar, monitorar e pagar.

## Desvantagens para um MVP

- **Escalabilidade única:** a escala é feita pela aplicação toda. No caso do nosso sistema, a leitura tende a ser muito maior que a escrita, e não dá para ajustar a escalabilidade apenas para um destes.
- **Ponto único de falha:** uma falha na aplicação ou no banco derruba todas as funcionalidades.
- **Requerimentos travados:** trocar de framework, linguagem ou banco afeta o sistema inteiro, não é possível alterar sem afetar alguma outra parte da aplicação.
- **Trabalho em paralelo:** com times maiores, aumentam os conflitos de código e a dependência entre entregas.
- **Deploy:** um simples deploy de correção demora muito mais, já que mesmo que altere um “;”, ainda sim terá que subir o pacote de toda aplicação por inteira.

---

## 12-Factor App aplicado ao monolito

**I. Base de Código - Atende**
Há uma única base de código versionada no Git, e o mesmo código foi implantado em dois ambientes (local e EC2 na AWS).

**II. Dependências - Atende**
As bibliotecas (Flask, psycopg2, Gunicorn) são declaradas no `requirements.txt` e isoladas em um ambiente virtual (`venv`) na EC2.

**III. Configurações - Parcial**
Com a implementação do serviço, o host, nome do banco, usuário e senha são lidos de variáveis de ambiente, sem credenciais no código. No deploy, as variáveis ficaram em `/etc/togglemaster.env` com permissão 600, fora do Git e direto na EC2. Porém sem a implementação do mesmo, era necessário realizar o “export” com essas informações

**IV. Serviços de Apoio - Atende**
O PostgreSQL é tratado como um recurso externo, acessado apenas por configuração. Isso permitiu trocar o banco local do Docker Compose pelo RDS sem alterar o código, apenas as variáveis de ambiente.

**V. Construa, lance, execute – Não atende**
Necessário pipeline de CI/CD que gere uma imagem versionada e a implante sempre da mesma forma, com possibilidade de rollback.

**VI. Processos - Atende**
A aplicação é stateless: não guarda sessão, cache ou arquivos em memória ou disco. Todo estado fica no banco.

**VII. Vínculo de porta – Não atende**
A porta 5000 está fixa no código (`app.run(..., port=5000)`). Para ajustar, é necessário ler a porta de uma variável (`PORT`), e colocar na frente um Load Balancer ou proxy reverso com HTTPS.

**VIII. Concorrência – Não atende**
A aplicação abre uma nova conexão com o banco a cada requisição (`get_db_connection` é chamada em todo handler), o que não escala bem e pode esgotar as conexões do RDS. Ajustes possíveis são: connection pooling (`psycopg2.pool` ou PgBouncer), ajuste do número de workers, ou até mesmo escalabilidade de instâncias.

**IX. Descartabilidade – Não atende**
Um ajuste possível seria health check que verifique o banco (hoje o `/health` só confirma que o processo responde).

**X. Dev/prod semelhantes – N/A**
Utilizado apenas um ambiente.

**XI. Logs - Parcial**
A aplicação usa `print`, que escreve em stdout e, portanto, respeita o princípio de tratar logs como fluxo de eventos. Mas não há níveis, timestamps, identificador de requisição nem formato estruturado. Poderia ser utilizado o CloudWatch

**XII. Processos de Admin - Atende**
A criação da tabela é um comando pontual (`flask init-db`) que roda com o mesmo código e as mesmas variáveis da aplicação.

---

## Diagrama

Link do diagrama no Miro:
https://miro.com/app/board/uXjVETe-kdo=/?share_link_id=749016279737

<img width="884" height="509" alt="image" src="https://github.com/user-attachments/assets/b58f07f3-904c-4f1b-b39b-94b42f5e944d" />

