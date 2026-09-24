# **ESPECIFICAÇÃO TÉCNICA E SCHEMA DDL (SUPABASE)**

---

**Projeto:** Controle de Ponto Eletrônico & Gestão de Banco de Horas  
**Público Alvo / Executor:** Agente de IA / Google Jules

## **1\. CAMADA DE ARQUITETURA 🏗️**

Esta camada define as tecnologias base, restrições e infraestrutura sem compilação (No-Build).

* **Modelo No-Build:** Execução nativa no navegador sem Node.js, Webpack, Vite ou pacotes NPM. Uso exclusivo de HTML5, CSS3 Nativo e JavaScript Vanilla (ES6+ Modules).  
* **Backend serverless (Supabase):** Banco de dados PostgreSQL com API PostgREST e Row Level Security (RLS) habilitado.  
* **Dependências via CDN ESM:**  
  * **Estilização:** Pico.css (https://cdn.jsdelivr.net/npm/@picocss/pico@1/css/pico.min.css)  
  * **Cliente Supabase:** @supabase/supabase-js via CDN ESM (https://cdn.jsdelivr.net/npm/@supabase/supabase-js/+esm)  
  * **Ícones:** Lucide Icons (https://unpkg.com/lucide@latest). *Proibido o uso de emojis na interface.*  
* **Hospedagem & Suporte:** GitHub Pages / GitHub Repositories. Responsividade direcionada para Tablets (orientação paisagem e retrato) e Desktops.  
* **Impressão Nativa:** Uso de window.print() estilizado via CSS @media print para geração física/PDF de comprovantes de registro.

## **2\. CAMADA FUNCIONAL (REGRAS DE NEGÓCIO) 📐**

Define as regras de cálculo, políticas de auditoria e comportamentos do sistema.

* **Regra de Tolerância de Ponto:**  
  * Margem de tolerância de 15 minutos na entrada e 15 minutos na saída.  
  * Se a tolerância for ultrapassada, o sistema deve debitar ou creditar **apenas os minutos que excederem a tolerância de 15 minutos**.  
  * *Exemplo:* Atraso de 25 minutos na entrada → Débito de \$25 \- 15 \= 10\$ minutos no Banco de Horas.  
* **Limite Crítico do Banco de Horas (Justa Causa):**  
  * Compensação em proporção 1:1.  
  * Se o saldo do colaborador atingir ou ultrapassar **\-20.00 horas (-1200 minutos)**, a trigger do banco altera o status do funcionário para ALERTA\_JUSTA\_CAUSA e dispara notificação no painel do RH e Superior Direto.  
* **Ajustes Manuais e Audit Trail:**  
  * Inclusões ou alterações manuais só podem ser efetuadas pelo **Superior Direto**.  
  * Cada alteração gera um registro imutável com motivo, horário anterior, novo horário, ID, nome e cargo do responsável.

## **3\. CAMADA DE IMPLEMENTAÇÃO 💻**

Define a estrutura de arquivos no repositório, módulos de código e o script SQL para execução no Supabase.

### **3.1 Estrutura de Arquivos no Repositório**

`/`  
`├── index.html         # Totem de Registro de Ponto e Impressão de Ticket`  
`├── admin.html         # Painel do Gestor (Ajustes Manuais, Relatórios e Auditoria)`  
`├── css/`  
`│   └── custom.css    # Estilos customizados e regras para @media print`  
`├── js/`  
`│   ├── app.js        # Lógica da UI e manipulação dos eventos de tela`  
`│   ├── supabase.js   # Inicialização do cliente Supabase via CDN ESM`  
`│   └── calc.js       # Funções puras de cálculo de horas e tolerâncias`  
`├── backlog.md        # Histórico granular de tarefas mantido pelo agente`  
`└── README.md         # Documentação de uso do repositório`

### **3.2 Script DDL SQL Completo para o Supabase (PostgreSQL)**

`-- =============================================================================`  
`-- ESQUEMA COMPLETO PARA CONTROLE DE PONTO E BANCO DE HORAS (SUPABASE)`  
`-- =============================================================================`

`-- 1. CRIAÇÃO DOS ENUMS`  
`CREATE TYPE user_role AS ENUM ('FUNCIONARIO', 'SUPERIOR_DIRETO', 'RH', 'ADMIN');`  
`CREATE TYPE status_funcionario AS ENUM ('ATIVO', 'INATIVO', 'ALERTA_JUSTA_CAUSA');`  
`CREATE TYPE tipo_marcacao AS ENUM ('ENTRADA', 'SAIDA_ALMOCO', 'RETORNO_ALMOCO', 'SAIDA');`  
`CREATE TYPE tipo_origem_registro AS ENUM ('TOTEM_CATRACA', 'AJUSTE_MANUAL', 'WEB_APP');`

`-- 2. TABELA DE FUNCIONÁRIOS / USUÁRIOS`  
`CREATE TABLE IF NOT EXISTS public.funcionarios (`  
    `id UUID PRIMARY KEY DEFAULT gen_random_uuid(),`  
    `matricula VARCHAR(20) UNIQUE NOT NULL,`  
    `nome VARCHAR(100) NOT NULL,`  
    `cpf VARCHAR(14) UNIQUE NOT NULL,`  
    `email VARCHAR(100) UNIQUE NOT NULL,`  
    `cargo VARCHAR(50) NOT NULL,`  
    `departamento VARCHAR(50) NOT NULL,`  
    `role user_role DEFAULT 'FUNCIONARIO'::user_role NOT NULL,`  
    `status status_funcionario DEFAULT 'ATIVO'::status_funcionario NOT NULL,`  
    `superior_direto_id UUID REFERENCES public.funcionarios(id),`  
    `cartao_rfid VARCHAR(50) UNIQUE,`  
    `saldo_banco_horas_minutos INT DEFAULT 0 NOT NULL, -- Armazenado em minutos para precisão`  
    `criado_em TIMESTAMPTZ DEFAULT NOW() NOT NULL,`  
    `atualizado_em TIMESTAMPTZ DEFAULT NOW() NOT NULL`  
`);`

`-- 3. TABELA DE REGISTROS DE PONTO`  
`CREATE TABLE IF NOT EXISTS public.registros_ponto (`  
    `id UUID PRIMARY KEY DEFAULT gen_random_uuid(),`  
    `funcionario_id UUID NOT NULL REFERENCES public.funcionarios(id) ON DELETE CASCADE,`  
    `data_hora TIMESTAMPTZ DEFAULT NOW() NOT NULL,`  
    `tipo tipo_marcacao NOT NULL,`  
    `origem tipo_origem_registro DEFAULT 'TOTEM_CATRACA'::tipo_origem_registro NOT NULL,`  
    `hash_comprovante TEXT NOT NULL,`  
    `comprovante_enviado_email BOOLEAN DEFAULT FALSE,`  
    `criado_em TIMESTAMPTZ DEFAULT NOW() NOT NULL`  
`);`

`-- 4. TABELA DE AUDITORIA E AJUSTES MANUAIS (AUDIT TRAIL IMUTÁVEL)`  
`CREATE TABLE IF NOT EXISTS public.audit_trail_ajustes (`  
    `id UUID PRIMARY KEY DEFAULT gen_random_uuid(),`  
    `registro_ponto_id UUID REFERENCES public.registros_ponto(id),`  
    `funcionario_id UUID NOT NULL REFERENCES public.funcionarios(id),`  
    `responsavel_id UUID NOT NULL REFERENCES public.funcionarios(id),`  
    `responsavel_nome VARCHAR(100) NOT NULL,`  
    `responsavel_cargo VARCHAR(50) NOT NULL,`  
    `data_hora_anterior TIMESTAMPTZ,`  
    `data_hora_nova TIMESTAMPTZ NOT NULL,`  
    `motivo TEXT NOT NULL,`  
    `criado_em TIMESTAMPTZ DEFAULT NOW() NOT NULL`  
`);`

`-- 5. TABELA DE HISTÓRICO DE BANCO DE HORAS`  
`CREATE TABLE IF NOT EXISTS public.historico_banco_horas (`  
    `id UUID PRIMARY KEY DEFAULT gen_random_uuid(),`  
    `funcionario_id UUID NOT NULL REFERENCES public.funcionarios(id) ON DELETE CASCADE,`  
    `data_referencia DATE NOT NULL,`  
    `minutos_trabalhados INT DEFAULT 0 NOT NULL,`  
    `minutos_esperados INT DEFAULT 480 NOT NULL, -- 8 horas padrão`  
    `saldo_dia_minutos INT DEFAULT 0 NOT NULL,`  
    `saldo_acumulado_minutos INT DEFAULT 0 NOT NULL,`  
    `criado_em TIMESTAMPTZ DEFAULT NOW() NOT NULL,`  
    `UNIQUE(funcionario_id, data_referencia)`  
`);`

`-- 6. FUNCTION E TRIGGER PARA ALERTA DE JUSTA CAUSA (-20h = -1200 MINUTOS)`  
`CREATE OR REPLACE FUNCTION check_alerta_justa_causa()`  
`RETURNS TRIGGER AS $$`  
`BEGIN`  
    `IF NEW.saldo_banco_horas_minutos <= -1200 THEN`  
        `NEW.status := 'ALERTA_JUSTA_CAUSA'::status_funcionario;`  
    `ELSIF NEW.saldo_banco_horas_minutos > -1200 AND OLD.status = 'ALERTA_JUSTA_CAUSA' THEN`  
        `NEW.status := 'ATIVO'::status_funcionario;`  
    `END IF;`  
    `NEW.atualizado_em := NOW();`  
    `RETURN NEW;`  
`END;`  
`$$ LANGUAGE plpgsql;`

`CREATE TRIGGER trg_check_alerta_justa_causa`  
`BEFORE UPDATE OF saldo_banco_horas_minutos ON public.funcionarios`  
`FOR EACH ROW`  
`EXECUTE FUNCTION check_alerta_justa_causa();`

`-- 7. ATIVAR ROW LEVEL SECURITY (RLS)`  
`ALTER TABLE public.funcionarios ENABLE ROW LEVEL SECURITY;`  
`ALTER TABLE public.registros_ponto ENABLE ROW LEVEL SECURITY;`  
`ALTER TABLE public.audit_trail_ajustes ENABLE ROW LEVEL SECURITY;`  
`ALTER TABLE public.historico_banco_horas ENABLE ROW LEVEL SECURITY;`

`-- POLÍTICAS RLS BASE`  
`CREATE POLICY "Leitura de funcionarios publica ou autenticada" ON public.funcionarios FOR SELECT USING (true);`  
`CREATE POLICY "Insercao de marcacoes de ponto" ON public.registros_ponto FOR INSERT WITH CHECK (true);`  
`CREATE POLICY "Visualizacao de marcacoes de ponto" ON public.registros_ponto FOR SELECT USING (true);`

### **3.3 Conteúdo do Módulo de Cálculo (\`js/calc.js\`)**

`/**`  
 `* Calcula o saldo em minutos aplicando a regra de tolerância de 15 min.`  
 `* @param {number} minutosAtrasoOuExcesso`   
 `* @returns {number} Minutos a serem creditados ou debitados no banco.`  
 `*/`  
`export function calcularToleranciaPonto(minutosAtrasoOuExcesso) {`  
  `const TOLERANCIA = 15;`  
  `if (Math.abs(minutosAtrasoOuExcesso) <= TOLERANCIA) {`  
    `return 0; // Dentro da tolerância, sem impacto no banco`  
  `}`  
  `if (minutosAtrasoOuExcesso > TOLERANCIA) {`  
    `return minutosAtrasoOuExcesso - TOLERANCIA; // Hora extra líquida`  
  `}`  
  `return minutosAtrasoOuExcesso + TOLERANCIA; // Débito líquido (ex: -25 + 15 = -10 min)`  
`}`

`/**`  
 `* Verifica se o saldo acumulado em minutos atinge a trava de -20.00h.`  
 `* @param {number} saldoMinutos`   
 `* @returns {boolean}`  
 `*/`  
`export function verificarStatusJustaCausa(saldoMinutos) {`  
  `const LIMITE_MINUTOS = -20 * 60; // -1200 minutos`  
  `return saldoMinutos <= LIMITE_MINUTOS;`  
`}`

### **3.4 Conteúdo do Backlog Inicial (\`backlog.md\`)**

`# Backlog de Desenvolvimento e Alterações`

`## [Sprint 1] - Estrutura Base e Banco de Dados`  
`- [x] Criação da especificação técnica e script DDL SQL do Supabase. (2026-08-27)`  
``- [ ] Configuração do cliente Supabase (`js/supabase.js`).``  
``- [ ] Implementação das funções puras de cálculo de horas (`js/calc.js`).``  
``- [ ] Interface de marcação e impressão de ticket (`index.html`).``  
``- [ ] Painel do gestor, relatórios e auditoria (`admin.html`).``  
