# Melhores Práticas em Automação de Testes com Cypress

Este documento compila as principais melhores práticas extraídas do repositório "Boas práticas em automação de testes com Cypress", que aborda 10 más práticas comuns na escrita de testes automatizados e suas respectivas soluções.

## 📋 Índice

1. [Evitar Browser Testing](#1-evitar-browser-testing)
2. [Eliminar Duplicação de Código](#2-eliminar-duplicação-de-código)
3. [Prevenir Flaky Tests](#3-prevenir-flaky-tests)
4. [Evitar Hardcoded Assertions](#4-evitar-hardcoded-assertions)
5. [Reduzir Complexidade Desnecessária](#5-reduzir-complexidade-desnecessária)
6. [Evitar Page Objects](#6-evitar-page-objects)
7. [Proteger Dados Sensíveis](#7-proteger-dados-sensíveis)
8. [Otimizar Testes Lentos](#8-otimizar-testes-lentos)
9. [Eliminar Dependências entre Testes](#9-eliminar-dependências-entre-testes)
10. [Evitar Abstrações Erradas](#10-evitar-abstrações-erradas)

---

## 1. Evitar Browser Testing

### ❌ Má Prática
Testar comportamentos padrão do navegador em vez de focar na aplicação sendo desenvolvida.

**Exemplo problemático:**
```javascript
// Testando se um link redireciona para a URL correta
it('directs user to login page', () => {
  cy.contains('.nav a', 'Login').click()
  cy.url().should('be.equal', 'https://example.com/login')
})
```

### ✅ Boa Prática
Verificar apenas as propriedades dos elementos HTML, confiando no comportamento padrão dos navegadores.

**Solução:**
```javascript
// Verificando apenas as propriedades do elemento
it('has correct login link properties', () => {
  cy.contains('.nav a', 'Login')
    .should('have.attr', 'href', '/login')
    .and('not.have.attr', 'target')
})
```

### 🎯 Princípios
- Navegadores já são testados pelos seus desenvolvedores
- Foque em testar sua aplicação, não o navegador
- Verifique propriedades dos elementos em vez de navegar

---

## 2. Eliminar Duplicação de Código

### ❌ Má Prática
Repetir o mesmo código em múltiplos testes, tornando a manutenção difícil.

### ✅ Boas Práticas

#### **2.1 Usar Hooks (beforeEach)**
```javascript
describe('Search functionality', () => {
  beforeEach(() => {
    cy.intercept('GET', '**/search**').as('getStories')
    cy.visit('https://example.com')
    cy.wait('@getStories')
  })

  it('searches by typing and hitting enter', () => {
    // Teste específico aqui
  })
})
```

#### **2.2 Comandos Customizados**
```javascript
// cypress/support/commands.js
Cypress.Commands.add('search', term => {
  cy.get('input[type="text"]')
    .should('be.visible')
    .clear()
    .type(`${term}{enter}`)
})

// Uso nos testes
it('searches for term', () => {
  cy.search('frontend testing')
})
```

#### **2.3 Iteração com Arrays**
```javascript
const terms = ['reactjs', 'vuejs']

terms.forEach(term => {
  it(`searches for ${term}`, () => {
    cy.search(term)
    cy.assertResults()
  })
})
```

#### **2.4 Usar Funcionalidades do Cypress**
```javascript
// Em vez de múltiplos .check()
cy.get('input[type="checkbox"]').check()

// Em vez de usar lodash.times()
Cypress._.times(3, () => {
  // Ação repetitiva
})
```

### 🎯 Benefícios
- Facilita manutenção
- Reduz chance de bugs
- Melhora legibilidade
- Promove reutilização

---

## 3. Prevenir Flaky Tests

### ❌ Má Prática
Esperar por estados intermediários ou usar esperas fixas.

**Exemplo problemático:**
```javascript
it('shows loading and then results', () => {
  cy.search('term')
  cy.contains('Loading...').should('be.visible')
  cy.contains('Loading...').should('not.exist')
  // Teste pode falhar se loading for muito rápido
})
```

### ✅ Boas Práticas

#### **3.1 Aguardar Requisições**
```javascript
it('searches and shows results', () => {
  cy.intercept('GET', '**/search**').as('getStories')
  cy.search('term')
  cy.wait('@getStories')
  cy.get('.results').should('be.visible')
})
```

#### **3.2 Usar Fixtures para Isolamento**
```javascript
beforeEach(() => {
  cy.intercept('GET', '**/search**', { 
    fixture: 'stories.json' 
  }).as('getStories')
})
```

#### **3.3 Confiar nas Esperas Automáticas**
```javascript
// Cypress já espera automaticamente
// Remova cy.wait() desnecessários
it('shows results', () => {
  cy.search('term')
  cy.get('.results').should('be.visible') // Espera automática
})
```

### 🎯 Princípios
- Evite esperar por estados intermediários
- Use fixtures para isolar do backend
- Confie nas esperas automáticas do Cypress
- Aguarde por requisições específicas quando necessário

---

## 4. Evitar Hardcoded Assertions

### ❌ Má Prática
Usar valores fixos nas verificações que podem quebrar quando dados mudam.

**Exemplo problemático:**
```javascript
it('shows search results', () => {
  cy.search('term')
  cy.get('.table-row').should('have.length', 2) // Valor fixo
})
```

### ✅ Boa Prática
Usar dados das fixtures nas verificações.

**Solução:**
```javascript
it('shows search results', () => {
  cy.fixture('stories').then(({ hits }) => {
    cy.search('term')
    cy.get('.table-row').should('have.length', hits.length)
    
    hits.forEach((story, index) => {
      cy.get('.table-row')
        .eq(index)
        .should('contain', story.title)
    })
  })
})
```

**Alternativa com desestruturação:**
```javascript
it('shows search results', { hits: require('../fixtures/stories.json').hits }, () => {
  cy.search('term')
  cy.get('.table-row').should('have.length', hits.length)
})
```

### 🎯 Benefícios
- Testes adaptáveis a mudanças nos dados
- Verificações dinâmicas e precisas
- Melhor manutenibilidade

---

## 5. Reduzir Complexidade Desnecessária

### ❌ Má Prática
Escrever código complexo quando existem comandos simples disponíveis.

**Exemplo problemático:**
```javascript
it('checks checkbox', () => {
  cy.get('#checkbox').then($checkbox => {
    if (!$checkbox.is(':checked')) {
      cy.get('#checkbox').click()
    }
  })
})
```

### ✅ Boa Prática
Usar comandos específicos do Cypress.

**Solução:**
```javascript
it('checks checkbox', () => {
  cy.get('#checkbox').check() // Simples e direto
})
```

### 🎯 Princípios
- Conheça os comandos disponíveis no Cypress
- Prefira simplicidade sobre complexidade
- Use comandos específicos para ações específicas

---

## 6. Evitar Page Objects

### ❌ Má Prática
Usar o padrão Page Objects desnecessariamente no Cypress.

**Exemplo problemático:**
```javascript
// editDestinationPage.js
class EditDestinationPage {
  updateInfo(destination) {
    // Código complexo aqui
  }
}

// Teste
const editPage = require('../page-objects/editDestinationPage')
editPage.updateInfo('New Destination')
```

### ✅ Boa Prática
Usar comandos customizados.

**Solução:**
```javascript
// cypress/support/commands.js
Cypress.Commands.add('updateDestination', destination => {
  // Implementação direta
  cy.get('#destination-input').clear().type(destination)
  cy.get('#save-button').click()
})

// Teste
it('updates destination', () => {
  cy.updateDestination('New Destination')
})
```

### 🎯 Vantagens dos Comandos Customizados
- Menos código necessário
- Disponíveis globalmente via `cy`
- Não precisam ser importados
- Permitem App Actions além de interações GUI
- Melhor integração com Cypress

---

## 7. Proteger Dados Sensíveis

### ❌ Má Prática
Versionar dados sensíveis no código.

**Exemplo problemático:**
```javascript
it('logs in user', () => {
  cy.get('#email').type('user@example.com') // Hardcoded
  cy.get('#password').type('secret123') // Hardcoded
})
```

### ✅ Boas Práticas

#### **7.1 Usar Variáveis de Ambiente**
```json
// cypress.env.json (não versionado)
{
  "user_email": "user@example.com",
  "user_password": "secret123"
}
```

```javascript
it('logs in user', () => {
  cy.get('#email').type(Cypress.env('user_email'))
  cy.get('#password', { log: false }).type(Cypress.env('user_password'))
})
```

#### **7.2 Configurar .gitignore**
```gitignore
cypress.env.json
cypress/downloads/
cypress/screenshots/
cypress/videos/
```

### 🎯 Práticas de Segurança
- Nunca versione credenciais
- Use `{ log: false }` para senhas
- Configure variáveis de ambiente
- Use .gitignore adequadamente

---

## 8. Otimizar Testes Lentos

### ❌ Más Práticas
- Usar `cy.wait()` com tempo fixo
- Navegar desnecessariamente
- Não isolar do backend

### ✅ Boas Práticas

#### **8.1 Evitar Esperas Fixas**
```javascript
// ❌ Ruim
cy.wait(3000)

// ✅ Bom - Timeout específico
cy.get('#element', { timeout: 10000 }).should('be.visible')
```

#### **8.2 Visitar Páginas Diretamente**
```javascript
// ❌ Navegação desnecessária
cy.visit('/')
cy.get('#signup-link').click()

// ✅ Visita direta
cy.visit('/signup')
```

#### **8.3 Mockar APIs**
```javascript
beforeEach(() => {
  cy.intercept('GET', '**/api/data**', { fixture: 'data.json' })
})
```

#### **8.4 Configurar Timeouts Inteligentes**
```javascript
// cypress.config.js
module.exports = defineConfig({
  e2e: {
    defaultCommandTimeout: 8000,
    video: false // Desabilita vídeos para acelerar
  }
})
```

### 🎯 Estratégias de Otimização
- Use fixtures para isolar do backend
- Visite páginas diretamente
- Configure timeouts apropriados
- Evite esperas fixas
- Desabilite recursos desnecessários

---

## 9. Eliminar Dependências entre Testes

### ❌ Má Prática
Criar testes que dependem uns dos outros.

**Exemplo problemático:**
```javascript
describe('Products CRUD', () => {
  it('creates product', () => { /* cria produto */ })
  it('reads product', () => { /* lê produto do teste anterior */ })
  it('updates product', () => { /* atualiza produto do teste anterior */ })
  it('deletes product', () => { /* deleta produto do teste anterior */ })
})
```

### ✅ Boa Prática
Consolidar em um único teste independente.

**Solução:**
```javascript
describe('Products CRUD', () => {
  it('performs complete CRUD operations', () => {
    // Create
    cy.createProduct('Test Product')
    cy.get('[data-cy="product"]').should('contain', 'Test Product')
    
    // Read
    cy.get('[data-cy="product"]').should('be.visible')
    
    // Update
    cy.updateProduct('Updated Product')
    cy.get('[data-cy="product"]').should('contain', 'Updated Product')
    
    // Delete
    cy.deleteProduct()
    cy.get('[data-cy="product"]').should('not.exist')
  })
})
```

### 🎯 Benefícios
- Testes independentes e isolados
- Execução mais rápida (menos setup)
- Falhas não afetam outros testes
- Melhor paralelização

---

## 10. Evitar Abstrações Erradas

### ❌ Má Prática
Abstrair verificações importantes em comandos customizados.

**Exemplo problemático:**
```javascript
// cypress/support/commands.js
Cypress.Commands.add('assertResults', () => {
  cy.get('.table-row').then(rows => {
    expect(rows.length).to.be.at.least(1)
  })
})

// Teste - não fica claro o que está sendo verificado
it('shows results', () => {
  cy.search('term')
  cy.assertResults() // O que exatamente está sendo verificado?
})
```

### ✅ Boa Prática
Manter verificações explícitas nos testes.

**Solução:**
```javascript
it('shows search results', () => {
  cy.search('term')
  
  // Verificações explícitas e claras
  cy.get('.table-row')
    .should('have.length.at.least', 1)
    .and('be.visible')
})

// Ou usando verificação implícita
it('shows search results', () => {
  cy.search('term')
  cy.get('.table-row').its('length').should('be.at.least', 1)
})
```

### 🎯 Princípios
- Prefira clareza sobre abstração em verificações
- É aceitável ter duplicação se melhora a legibilidade
- Abstraia ações, mantenha verificações explícitas
- Testes devem ser auto-documentados

---

## 🛠️ Configuração do Projeto

### Estrutura Recomendada
```
project/
├── cypress/
│   ├── e2e/
│   ├── fixtures/
│   ├── support/
│   │   ├── commands.js
│   │   └── e2e.js
├── cypress.config.js
├── cypress.env.json (não versionado)
├── package.json
└── .gitignore
```

### Configuração Básica
```javascript
// cypress.config.js
const { defineConfig } = require('cypress')

module.exports = defineConfig({
  e2e: {
    baseUrl: 'https://your-app.com',
    viewportWidth: 1280,
    viewportHeight: 720,
    defaultCommandTimeout: 8000,
    video: false,
    screenshotOnRunFailure: true
  }
})
```

### Dependências Essenciais
```json
{
  "devDependencies": {
    "cypress": "^12.8.1",
    "@faker-js/faker": "^7.6.0"
  }
}
```

---

## 📚 Recursos Adicionais

### Scripts Úteis
```json
{
  "scripts": {
    "test": "cypress run",
    "test:headed": "cypress run --headed",
    "cy:open": "cypress open",
    "test:spec": "cypress run --spec"
  }
}
```

### Comandos de Execução
```bash
# Executar todos os testes
npm test

# Executar teste específico
npx cypress run --spec cypress/e2e/login.cy.js

# Abrir interface interativa
npm run cy:open

# Executar com cabeçalho visível
npm run test:headed
```

---

## 🎯 Resumo das Melhores Práticas

1. **Foque na aplicação**, não no navegador
2. **Elimine duplicação** com hooks e comandos customizados
3. **Previna flaky tests** aguardando requisições específicas
4. **Use dados dinâmicos** em vez de valores hardcoded
5. **Simplifique** usando comandos específicos do Cypress
6. **Prefira comandos customizados** sobre Page Objects
7. **Proteja dados sensíveis** com variáveis de ambiente
8. **Otimize performance** mockando APIs e visitando diretamente
9. **Mantenha independência** entre testes
10. **Seja explícito** nas verificações importantes

### 🏆 Resultado
Seguindo essas práticas, você terá testes:
- Mais rápidos e confiáveis
- Fáceis de manter e entender
- Seguros e bem estruturados
- Independentes e escaláveis

---

*Este documento foi baseado no curso "Boas práticas em automação de testes com Cypress" da Escola Talking About Testing.*