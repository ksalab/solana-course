---
назва: Співставлення даних облікових записів
цілі:
- Пояснити ризики для безпеки, пов'язані з відсутністю перевірок валідності даних
- Реалізувати перевірку даних з використанням розгорнутої форми Rust
- Реалізувати перевірку даних за допомогою обмежень Anchor
---

# TL;DR

- Використовуйте **перевірки даних** для перевірки відповідності даних рахунку очікуваному значенню. Без відповідних перевірок даних в інструкції можуть бути використані неочікувані облікові записи.
- Щоб реалізувати перевірку даних у Rust, просто порівняйте дані, що зберігаються в обліковому записі, з очікуваним значенням.
    
    ```rust
    if ctx.accounts.user.key() != ctx.accounts.user_data.user {
        return Err(ProgramError::InvalidAccountData.into());
    }
    ```
    
- У Anchor ви можете використовувати `constraint` для перевірки того, що заданий вираз набуває значення true. Крім того, ви можете використовувати `has_one`, щоб перевірити, що поле цільового облікового запису, яке зберігається в обліковому записі, збігається з ключем облікового запису в структурі `Accounts`.

# Огляд

Перевірка відповідності даних облікового запису відноситься до перевірок валідності даних, які використовуються для перевірки відповідності даних, що зберігаються в обліковому записі, очікуваному значенню. Перевірка даних дає змогу включити додаткові обмеження, щоб гарантувати, що в інструкцію будуть передані відповідні рахунки. 

Це може бути корисно, коли облікові записи, яких вимагає команда, залежать від значень, що зберігаються в інших облікових записах, або якщо команда залежить від даних, що зберігаються в обліковому записі.

### Відсутня перевірка валідності даних

Наведений нижче приклад містить інструкцію `update_admin`, яка оновлює поле `admin`, що зберігається в обліковому записі `admin_config`. 

В інструкції відсутня перевірка перевірки даних, щоб переконатися, що обліковий запис `admin`, який підписує транзакцію, збігається з `admin`, що зберігається в обліковому записі `admin_config`. Це означає, що будь-який обліковий запис, який підписує транзакцію і передається в інструкції як обліковий запис `admin`, може оновити обліковий запис `admin_config`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### Додати перевірку валідності даних

Базовий підхід Rust до вирішення цієї проблеми полягає у простому порівнянні переданого ключа `admin` з ключем `admin`, що зберігається в обліковому записі `admin_config`, і видачі помилки, якщо вони не збігаються.

```rust
if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

Додавши перевірку даних, інструкція `update_admin` буде оброблятися тільки в тому випадку, якщо підписант транзакції `admin` збігається з `admin`, що зберігається в обліковому записі `admin_config`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
      if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
            return Err(ProgramError::InvalidAccountData.into());
        }
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### Використання обмежень Anchor

Anchor спрощує це за допомогою обмеження `has_one`. Ви можете використовувати обмеження `has_one`, щоб перенести перевірку даних з логіки інструкції до структури `UpdateAdmin`.

У наведеному нижче прикладі `has_one = admin` вказує, що обліковий запис `admin`, який підписує транзакцію, повинен відповідати полю `admin`, що зберігається в обліковому записі `admin_config`. Для використання обмеження `has_one` угода про іменування поля даних в обліковому записі повинна відповідати іменуванням у структурі перевірки облікового запису.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        has_one = admin
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

Крім того, ви можете використати `constraint` для ручного додавання виразу, який має набути значення true для продовження виконання. Це корисно, коли з якихось причин іменування не може бути послідовним або коли вам потрібен складніший вираз для повної перевірки вхідних даних.

```rust
#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        constraint = admin_config.admin == admin.key()
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}
```

# Лабораторна робота

У цій лабораторній роботі ми створимо просту програму "vault", подібну до програми, яку ми використовували в уроках "Авторизація підписувача" та "Перевірка власника". Подібно до цих лабораторних робіт, у цій лабораторній роботі ми покажемо, як перевірка відсутності даних може призвести до спустошення сховища.

### 1. Початковий етап

Для початку завантажте початковий код з гілки `starter` [цього репозиторію](https://github.com/Unboxed-Software/solana-account-data-matching). Стартовий код містить програму з двома інструкціями та шаблонним налаштуванням для тестового файлу. 

Інструкція `initialize_vault` ініціалізує новий обліковий запис `Vault` і новий `TokenAccount`. Обліковий запис `Vault` буде зберігати адресу облікового запису токенів, повноваження сховища та обліковий запис токенів призначення для виведення.

Повноваження нового облікового запису токенів буде встановлено як "vault", PDA програми. Це дозволить обліковому запису `vault` підписувати передачу токенів з облікового запису токенів. 

Інструкція `insecure_withdraw` передає всі токени з облікового запису токенів `vault` до облікового запису токенів `withdraw_destination`.

Зверніть увагу, що в цій інструкції **є**  перевірка підписувача для `authority` і перевірка власника для `vault`. Однак, ніде в логіці перевірки облікового запису або інструкції немає коду, який би перевіряв відповідність облікового запису `authority`, переданого в інструкцію, обліковому запису `authority` на `vault`.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        ctx.accounts.vault.withdraw_destination = ctx.accounts.withdraw_destination.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 32,
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
    withdraw_destination: Pubkey,
}
```

### 2. Тест інструкції `insecure_withdraw`

Щоб довести, що це проблема, напишемо тест, в якому обліковий запис, відмінний від `authority` сховища, намагається вивести кошти зі сховища.

Файл тесту містить код для виклику інструкції `initialize_vault` з використанням гаманця провайдера в якості `authority`, а потім карбує 100 токенів на обліковий запис токенів `vault`.

Додайте тест для виклику інструкції `insecure_withdraw`. Використовуйте `withdrawDestinationFake` як акаунт `withdrawDestination` і `walletFake` як `authority`. Потім відправте транзакцію, використовуючи `walletFake`.

Оскільки немає ніяких перевірок на відповідність переданого в інструкції акаунта `authority` значенням, що зберігаються на акаунті `vault`, ініціалізованому в першому тесті, інструкція буде успішно оброблена і токени будуть переведені на акаунт `withdrawDestinationFake`.

```tsx
describe("account-data-matching", () => {
  ...
  it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestinationFake,
        authority: walletFake.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть `anchor test`, щоб переконатися, що обидві транзакції завершаться успішно.

```bash
account-data-matching
  ✔ Initialize Vault (811ms)
  ✔ Insecure withdraw (403ms)
```

### 3. Додаємо інструкцію `secure_withdraw`

Давайте реалізуємо безпечну версію цієї інструкції під назвою `secure_withdraw`.

Ця інструкція буде ідентична інструкції `insecure_withdraw`, за винятком того, що ми будемо використовувати обмеження `has_one` в структурі перевірки облікового запису (`SecureWithdraw`), щоб перевірити, що обліковий запис `authority`, переданий в інструкції, відповідає обліковому запису `authority` в обліковому записі `vault`. Таким чином, тільки правильний обліковий запис може вилучати токени сховища.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;
    ...
    pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority,
        has_one = withdraw_destination,

    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

### 4. Тестування інструкції `secure_withdraw`

Тепер протестуємо інструкцію `secure_withdraw` за допомогою двох тестів: з використанням `walletFake` в якості авторизації та з використанням `wallet` в якості авторизації. Ми очікуємо, що перший виклик поверне помилку, а другий буде успішним.

```tsx
describe("account-data-matching", () => {
  ...
  it("Secure withdraw, expect error", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenPDA,
          withdrawDestination: withdrawDestinationFake,
          authority: walletFake.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })

  it("Secure withdraw", async () => {
    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      tokenPDA,
      wallet.payer,
      100
    )

    await program.methods
      .secureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestination,
        authority: wallet.publicKey,
      })
      .rpc()

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть `anchor test`, щоб побачити, що транзакція з використанням невірного облікового запису тепер повертає помилку Anchor Error, тоді як транзакція з використанням правильних облікових записів завершується успішно.

```bash
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: ConstraintHasOne. Error Number: 2001. Error Message: A has one constraint was violated.',
'Program log: Left:',
'Program log: DfLZV18rD7wCQwjYvhTFwuvLh49WSbXFeJFPQb5czifH',
'Program log: Right:',
'Program log: 5ovvmG5ntwUC7uhNWfirjBHbZD96fwuXDMGXiyMwPg87',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 10401 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d1'
```

Зверніть увагу, що Anchor вказує в логах обліковий запис, який спричиняє помилку (`AnchorError caused by account: vault`).

```bash
✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
```

І ось так просто ви закрили лазівку в безпеці. Більшість з цих потенційних уразливостей є досить простими. Однак, по мірі того, як ваші програми зростають в обсязі та складності, стає все легше і легше пропустити можливі уразливості. Дуже корисно мати звичку писати тести, які посилають інструкції, що "не повинні" працювати. Чим більше, тим краще. Таким чином ви зможете виявити проблеми ще до розгортання.

Якщо ви хочете поглянути на остаточний код рішення, ви можете знайти його на гілці `solution` у [репозиторії](https://github.com/Unboxed-Software/solana-account-data-matching/tree/solution).

# Завдання

Як і в інших уроках цього розділу, ваша можливість попрактикуватися в уникненні цієї уразливості полягає в аудиті власних або інших програм.

Витратьте трохи часу на перевірку принаймні однієї програми і переконайтеся, що у ній передбачено належну перевірку даних для уникнення вразливостей у безпеці.

Пам'ятайте, якщо ви знайшли помилку або уразливість у чужій програмі, будь ласка, повідомте про це! Якщо ви знайшли помилку у власній програмі, не забудьте негайно виправити її.

## Завершили завдання?

Завантажте свій код на GitHub і [розкажіть нам, що ви думаєте про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=a107787e-ad33-42bb-96b3-0592efc1b92f)
