# Page Objects: Quando Usar e Quando Não Usar

## 📖 O que são Page Objects?

Page Objects é um padrão de design que encapsula elementos e ações de uma página web em uma classe ou objeto, promovendo reutilização e manutenibilidade do código de testes.

### Exemplo Clássico de Page Object:
```javascript
// pages/LoginPage.js
class LoginPage {
  constructor() {
    this.emailField = '#email'
    this.passwordField = '#password'
    this.submitButton = '#login-btn'
    this.errorMessage = '.error-msg'
  }

  visit() {
    cy.visit('/login')
  }

  fillEmail(email) {
    cy.get(this.emailField).type(email)
  }

  fillPassword(password) {
    cy.get(this.passwordField).type(password)
  }

  submit() {
    cy.get(this.submitButton).click()
  }

  login(email, password) {
    this.fillEmail(email)
    this.fillPassword(password)
    this.submit()
  }

  getErrorMessage() {
    return cy.get(this.errorMessage)
  }
}

export default LoginPage
```

---

## ❌ Quando NÃO Usar Page Objects

### 1. **No Cypress (Recomendação Oficial)**

O time do Cypress **desencoraja** o uso de Page Objects. Aqui está o porquê:

#### **Problemas com Page Objects no Cypress:**

##### **1.1 Complexidade Desnecessária**
```javascript
// ❌ Page Object - Muito código para pouco valor
class LoginPage {
  constructor() {
    this.emailField = '#email'
    this.passwordField = '#password'
    this.submitButton = '#login-btn'
  }

  login(email, password) {
    cy.get(this.emailField).type(email)
    cy.get(this.passwordField).type(password)
    cy.get(this.submitButton).click()
  }
}

// Uso no teste
const loginPage = new LoginPage()
loginPage.login('user@test.com', 'password')

// ✅ Comando Customizado - Mais simples
Cypress.Commands.add('login', (email, password) => {
  cy.get('#email').type(email)
  cy.get('#password').type(password)
  cy.get('#login-btn').click()
})

// Uso no teste
cy.login('user@test.com', 'password')
```

##### **1.2 Necessidade de Imports**
```javascript
// ❌ Page Objects requerem imports
import LoginPage from '../pages/LoginPage'
const loginPage = new LoginPage()

// ✅ Comandos customizados são globais
cy.login() // Disponível automaticamente
```

##### **1.3 Limitação para GUI apenas**
```javascript
// ❌ Page Objects focam apenas em GUI
class LoginPage {
  login(email, password) {
    cy.get('#email').type(email)
    cy.get('#password').type(password)
    cy.get('#login-btn').click()
  }
}

// ✅ Comandos customizados permitem App Actions
Cypress.Commands.add('login', (email, password) => {
  // Opção 1: GUI (para testes de UI)
  cy.get('#email').type(email)
  cy.get('#password').type(password)
  cy.get('#login-btn').click()
})

Cypress.Commands.add('loginViaAPI', (email, password) => {
  // Opção 2: API (para setup rápido)
  cy.request({
    method: 'POST',
    url: '/api/login',
    body: { email, password }
  }).then(response => {
    window.localStorage.setItem('authToken', response.body.token)
  })
})
```

### 2. **Em Projetos Pequenos**
- Overhead desnecessário
- Poucos elementos para gerenciar
- Mudanças frequentes na UI

### 3. **Quando a UI Muda Constantemente**
- Page Objects se tornam obsoletos rapidamente
- Manutenção constante necessária
- Melhor usar seletores diretos

### 4. **Para Testes de API**
- Page Objects não fazem sentido
- Foque em requests/responses
- Use fixtures e comandos customizados

---

## ✅ Quando Usar Page Objects

### 1. **Selenium/WebDriver (Não Cypress)**

Page Objects fazem mais sentido em ferramentas como Selenium:

```java
// Java + Selenium - Faz sentido usar Page Objects
public class LoginPage {
    private WebDriver driver;
    
    @FindBy(id = "email")
    private WebElement emailField;
    
    @FindBy(id = "password")
    private WebElement passwordField;
    
    public void login(String email, String password) {
        emailField.sendKeys(email);
        passwordField.sendKeys(password);
        // ...
    }
}
```

### 2. **Aplicações Grandes e Estáveis**
- UI bem estabelecida
- Muitas páginas com elementos similares
- Equipe grande que precisa de padronização

### 3. **Quando Há Muito Reuso**
```javascript
// Se você tem MUITA reutilização similar
class FormPage {
  fillForm(data) {
    Object.keys(data).forEach(field => {
      cy.get(`#${field}`).type(data[field])
    })
  }
  
  submitForm() {
    cy.get('#submit').click()
  }
  
  validateForm() {
    cy.get('.success').should('be.visible')
  }
}
```

### 4. **Testes de Múltiplos Frameworks**
- Quando você precisa suportar diferentes ferramentas
- Abstração ajuda na portabilidade

---

## 🎯 Alternativas Recomendadas para Cypress

### 1. **Comandos Customizados**
```javascript
// cypress/support/commands.js
Cypress.Commands.add('login', (email, password) => {
  cy.visit('/login')
  cy.get('#email').type(email)
  cy.get('#password').type(password)
  cy.get('#login-btn').click()
})

Cypress.Commands.add('createProduct', (product) => {
  cy.get('#product-name').type(product.name)
  cy.get('#product-price').type(product.price)
  cy.get('#save-btn').click()
})
```

### 2. **App Actions**
```javascript
// Ações que manipulam o estado da aplicação
Cypress.Commands.add('seedDatabase', (data) => {
  cy.task('db:seed', data)
})

Cypress.Commands.add('loginViaAPI', (credentials) => {
  cy.request('POST', '/api/auth', credentials)
    .then(response => {
      cy.window().its('localStorage').invoke('setItem', 'token', response.body.token)
    })
})
```

### 3. **Helpers e Utilities**
```javascript
// cypress/support/utils.js
export const selectors = {
  login: {
    email: '#email',
    password: '#password',
    submit: '#login-btn'
  },
  product: {
    name: '#product-name',
    price: '#product-price',
    save: '#save-btn'
  }
}

// Uso nos testes
import { selectors } from '../support/utils'

it('logs in user', () => {
  cy.get(selectors.login.email).type('user@test.com')
  cy.get(selectors.login.password).type('password')
  cy.get(selectors.login.submit).click()
})
```

---

## 🔄 Migração de Page Objects para Comandos Customizados

### Antes (Page Object):
```javascript
// pages/LoginPage.js
class LoginPage {
  visit() {
    cy.visit('/login')
  }

  fillCredentials(email, password) {
    cy.get('#email').type(email)
    cy.get('#password').type(password)
  }

  submit() {
    cy.get('#submit').click()
  }

  login(email, password) {
    this.visit()
    this.fillCredentials(email, password)
    this.submit()
  }
}

// Teste
import LoginPage from '../pages/LoginPage'
const loginPage = new LoginPage()

it('should login', () => {
  loginPage.login('user@test.com', 'password')
  cy.url().should('include', '/dashboard')
})
```

### Depois (Comando Customizado):
```javascript
// cypress/support/commands.js
Cypress.Commands.add('login', (email, password) => {
  cy.visit('/login')
  cy.get('#email').type(email)
  cy.get('#password').type(password)
  cy.get('#submit').click()
})

// Teste
it('should login', () => {
  cy.login('user@test.com', 'password')
  cy.url().should('include', '/dashboard')
})
```

---

## 📊 Comparação: Page Objects vs Comandos Customizados

| Aspecto | Page Objects | Comandos Customizados |
|---------|-------------|----------------------|
| **Linhas de código** | Mais (classe + imports) | Menos (função direta) |
| **Disponibilidade** | Precisa importar | Global via `cy` |
| **Flexibilidade** | Limitado a GUI | GUI + API + Tasks |
| **Manutenção** | Mais complexa | Mais simples |
| **Curva de aprendizado** | Maior | Menor |
| **Performance** | Overhead de classes | Direto |
| **Debugging** | Mais difícil | Mais fácil |

---

## 🛠️ Implementação Prática

### Estrutura Recomendada (Sem Page Objects):
```
cypress/
├── e2e/
│   ├── auth/
│   ├── products/
│   └── user-management/
├── fixtures/
├── support/
│   ├── commands.js          # Comandos customizados
│   ├── utils.js            # Utilitários e seletores
│   └── e2e.js              # Setup global
└── tasks/                  # App Actions
```

### Exemplo de Comando Avançado:
```javascript
// cypress/support/commands.js
Cypress.Commands.add('loginAs', (userType = 'standard') => {
  const users = {
    admin: { email: 'admin@test.com', password: 'admin123' },
    standard: { email: 'user@test.com', password: 'user123' },
    guest: { email: 'guest@test.com', password: 'guest123' }
  }

  const user = users[userType]
  
  // Opção rápida via API
  if (Cypress.env('FAST_LOGIN')) {
    cy.request('POST', '/api/auth', user)
      .then(response => {
        cy.window().its('localStorage').invoke('setItem', 'token', response.body.token)
        cy.visit('/dashboard')
      })
  } else {
    // Opção GUI para testes de interface
    cy.visit('/login')
    cy.get('#email').type(user.email)
    cy.get('#password').type(user.password)
    cy.get('#login-btn').click()
  }
})
```

---

## 🎯 Resumo e Recomendações

### ❌ **Evite Page Objects quando:**
- Usando Cypress (recomendação oficial)
- Projeto pequeno/médio
- UI muda frequentemente
- Foco em simplicidade

### ✅ **Use Page Objects quando:**
- Usando Selenium/WebDriver
- Aplicação muito grande e estável
- Equipe grande precisa de padronização
- Múltiplas ferramentas de teste

### 🎯 **Para Cypress, prefira:**
1. **Comandos Customizados** para ações
2. **App Actions** para setup de estado
3. **Fixtures** para dados de teste
4. **Utilities** para seletores e helpers

### 💡 **Lembre-se:**
> "A melhor abstração é aquela que resolve um problema real sem criar complexidade desnecessária"

No Cypress, comandos customizados oferecem todos os benefícios dos Page Objects com muito menos complexidade e muito mais flexibilidade.

---

*Baseado nas recomendações oficiais do Cypress e melhores práticas da comunidade de testes.*