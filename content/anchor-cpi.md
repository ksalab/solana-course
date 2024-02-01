---
назва: Закріпити CPI та помилки
цілі:
- Створення міжпрограмних викликів (CPI) з якірної програми
- Використовуйте функцію `cpi` для створення допоміжних функцій для виклику інструкцій з існуючих якірних програм
- Використовуйте `invoke` та `invoke_signed` для створення CPI там, де допоміжні функції CPI недоступні
- Створення та повернення користувацьких помилок Anchor
---

# TL;DR

- Anchor надає спрощений спосіб створення CPI за допомогою **`CpiContext`**
- Функція **`cpi`** Anchor генерує допоміжні функції CPI для виклику інструкцій у існуючих програмах Anchor
- Якщо у вас немає доступу до допоміжних функцій CPI, ви можете використовувати `invoke` та `invoke_signed` безпосередньо
- Макрос атрибута **`error_code`** використовується для створення користувацьких помилок Anchor

# Огляд

Якщо ви згадаєте [перший урок з CPI](https://github.com/Unboxed-Software/solana-course/blob/main/content/cpi), то пригадаєте, що побудова CPI у ванільному Rust може бути складним завданням. Однак Anchor робить це трохи простішим, особливо якщо програма, яку ви викликаєте, також є програмою Anchor, до якої ви маєте доступ.

У цьому уроці ви дізнаєтеся, як створити Anchor CPI. Ви також дізнаєтеся, як виводити власні помилки з якірної програми, щоб ви могли почати писати більш складні якірні програми.

## Міжпрограмні виклики (CPI) у Anchor

Нагадуємо, що CPI дозволяють програмам викликати інструкції з інших програм за допомогою функцій `invoke` або `invoke_signed`. Це дозволяє створювати нові програми на основі існуючих (ми називаємо це компонуванням).

Хоча створення CPI безпосередньо за допомогою `invoke` або `invoke_signed` все ще залишається можливим, Anchor також надає спрощений спосіб створення CPI за допомогою `CpiContext`.

У цьому уроці ви скористаєтеся скринькою `anchor_spl` для створення CPI до програми токенів SPL. Ви можете [ознайомитися з можливостями пакунка `anchor_spl`](https://docs.rs/anchor-spl/latest/anchor_spl/#).

### `CpiContext`

Першим кроком у створенні CPI є створення екземпляра `CpiContext`. `CpiContext` дуже схожий на `Context`, перший тип аргументу, необхідний для функцій інструкції Anchor. Вони обидва оголошені в одному модулі і мають схожу функціональність.

Тип `CpiContext` визначає неаргументні вхідні дані для міжпрограмних викликів:

- `accounts` - список рахунків, необхідних для виклику інструкції
- `remaining_accounts` - всі рахунки, що залишилися
- `program` - ідентифікатор програми, що викликається
- `cigner_seeds` - якщо підписує КПК, включити семантику, необхідну для отримання КПК

```rust
pub struct CpiContext<'a, 'b, 'c, 'info, T>
where
    T: ToAccountMetas + ToAccountInfos<'info>,
{
    pub accounts: T,
    pub remaining_accounts: Vec<AccountInfo<'info>>,
    pub program: AccountInfo<'info>,
    pub signer_seeds: &'a [&'b [&'c [u8]]],
}
```

Ви використовуєте `CpiContext::new` для створення нового екземпляра при передачі оригінального підпису транзакції.

```rust
CpiContext::new(cpi_program, cpi_accounts)
```

```rust
pub fn new(
        program: AccountInfo<'info>,
        accounts: T
    ) -> Self {
    Self {
        accounts,
        program,
        remaining_accounts: Vec::new(),
        signer_seeds: &[],
    }
}
```

Ви використовуєте `CpiContext::new_with_signer` для створення нового екземпляра при підписанні від імені PDA для CPI.

```rust
CpiContext::new_with_signer(cpi_program, cpi_accounts, seeds)
```

```rust
pub fn new_with_signer(
    program: AccountInfo<'info>,
    accounts: T,
    signer_seeds: &'a [&'b [&'c [u8]]],
) -> Self {
    Self {
        accounts,
        program,
        signer_seeds,
        remaining_accounts: Vec::new(),
    }
}
```

### Акаунти CPI

Однією з головних особливостей `CpiContext`, яка спрощує міжпрограмні виклики, є те, що аргумент `accounts` є узагальненим типом, який дозволяє вам передавати будь-який об'єкт, що приймає ознаки `ToAccountMetas` і `ToAccountInfos<'info>`.

Ці ознаки додаються за допомогою макросу атрибута `#[derive(Accounts)]`, який ви використовували раніше при створенні структур для представлення рахунків інструкцій. Це означає, що ви можете використовувати подібні структури з `CpiContext`.

Це допомагає з організацією коду та безпекою типів.

### Виклик інструкції з іншої програми-якоря

Якщо програма, яку ви викликаєте, є Anchor-програмою з опублікованим каркасом, Anchor може генерувати для вас будівники інструкцій та допоміжні функції CPI.

Просто оголосіть залежність вашої програми від програми, яку ви викликаєте, у файлі `Cargo.toml` вашої програми наступним чином:

```
[dependencies]
callee = { path = "../callee", features = ["cpi"]}
```

Додавши `features = ["cpi"]`, ви вмикаєте функцію `cpi` і ваша програма отримує доступ до модуля `callee::cpi`.

Модуль `cpi` виводить інструкції `callee` у вигляді функції Rust, яка приймає як аргументи `CpiContext` і будь-які додаткові дані інструкції. Ці функції використовують той самий формат, що і функції інструкцій у ваших якірних програмах, тільки з `CpiContext` замість `Context`. Модуль `cpi` також розкриває структури рахунків, необхідні для виклику інструкцій.

Наприклад, якщо `callee` має інструкцію `do_something`, яка вимагає облікових записів, визначених у структурі `DoSomething`, ви можете викликати `do_something` наступним чином:

```rust
use anchor_lang::prelude::*;
use callee;
...

#[program]
pub mod lootbox_program {
    use super::*;

    pub fn call_another_program(ctx: Context<CallAnotherProgram>, params: InitUserParams) -> Result<()> {
        callee::cpi::do_something(
            CpiContext::new(
                ctx.accounts.callee.to_account_info(),
                callee::DoSomething {
                    user: ctx.accounts.user.to_account_info()
                }
            )
        )
        Ok(())
    }
}
...
```

### Виклик інструкції у програмі, яка не є Anchor

Якщо програма, яку ви викликаєте, не є *не* Anchor програмою, є два можливі варіанти:

1. Можливо, що розробники програми опублікували скриньку з власними допоміжними функціями для виклику у своїй програмі. Наприклад, ящик `anchor_spl` надає допоміжні функції, які практично ідентичні з точки зору місця виклику тим, що ви отримаєте за допомогою модуля `cpi` у програмі Anchor. Наприклад, ви можете карбувати за допомогою допоміжної функції [`mint_to`](https://docs.rs/anchor-spl/latest/src/anchor_spl/token.rs.html#36-58) і використовувати структуру [`MintTo` accounts](https://docs.rs/anchor-spl/latest/anchor_spl/token/struct.MintTo.html).

    ```rust
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::MintTo {
                mint: ctx.accounts.mint_account.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint_authority").unwrap()],
            ]]
        ),
        amount,
    )?;
    ```
    
2. Якщо для програми, інструкції якої вам потрібно викликати, не існує допоміжного модуля, ви можете повернутися до використання `invoke` і `invoke_signed`. Насправді, у вихідному коді допоміжної функції `mint_to`, на яку ми посилалися вище, показано приклад використання `invoke_signed`, коли задано `CpiContext`. Ви можете слідувати подібному шаблону, якщо вирішите використовувати структуру рахунків і `CpiContext` для організації та підготовки вашого CPI.

    ```rust
    pub fn mint_to<'a, 'b, 'c, 'info>(
        ctx: CpiContext<'a, 'b, 'c, 'info, MintTo<'info>>,
        amount: u64,
    ) -> Result<()> {
        let ix = spl_token::instruction::mint_to(
            &spl_token::ID,
            ctx.accounts.mint.key,
            ctx.accounts.to.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?;
        solana_program::program::invoke_signed(
            &ix,
            &[
                ctx.accounts.to.clone(),
                ctx.accounts.mint.clone(),
                ctx.accounts.authority.clone(),
            ],
            ctx.signer_seeds,
        )
        .map_err(Into::into)
    }
    ```

## Додавання помилок в Anchor

На даний момент ми достатньо глибоко вивчили Anchor, щоб знати, як створювати власні помилки.

Зрештою, всі програми повертають один і той самий тип помилки: [`ProgramError`](https://docs.rs/solana-program/latest/solana_program/program_error/enum.ProgramError.html). Однак, при написанні програми з використанням Anchor ви можете використовувати `AnchorError` як абстракцію над `ProgramError`. Ця абстракція надає додаткову інформацію у випадку збою програми, зокрема:

- Назва та номер помилки
- Місце в коді, де виникла помилка
- Обліковий запис, який порушив обмеження

```rust
pub struct AnchorError {
    pub error_name: String,
    pub error_code_number: u32,
    pub error_msg: String,
    pub error_origin: Option<ErrorOrigin>,
    pub compared_values: Option<ComparedValues>,
}
```

Помилки Anchor можна розділити на:

- Внутрішні помилки, які фреймворк повертає з власного коду
- Користувацькі помилки, які ви, розробник, можете створити

Ви можете додавати помилки, унікальні для вашої програми, використовуючи атрибут `error_code`. Просто додайте цей атрибут до кастомного типу `enum`. Після цього ви можете використовувати варіанти `enum` як помилки у вашій програмі. Крім того, ви можете додати повідомлення про помилку до кожного варіанту за допомогою атрибута `msg`. Клієнти можуть відображати це повідомлення у разі виникнення помилки.

```rust
#[error_code]
pub enum MyError {
    #[msg("MyAccount може зберігати лише дані, менші за 100")]
    DataTooLarge
}
```

Щоб повернути користувацьку помилку, ви можете скористатися макросом [err](https://docs.rs/anchor-lang/latest/anchor_lang/macro.err.html) або [error](https://docs.rs/anchor-lang/latest/anchor_lang/prelude/macro.error.html) з функції-інструкції. Вони додають інформацію про файл і рядок до помилки, яка потім записується Anchor, щоб допомогти вам з налагодженням.

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        if data.data >= 100 {
            return err!(MyError::DataTooLarge);
        }
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount може зберігати лише дані, менші за 100")]
    DataTooLarge
}
```

Крім того, ви можете використовувати макрос [require](https://docs.rs/anchor-lang/latest/anchor_lang/macro.require.html) для спрощення повернення помилок. Наведений вище код може бути рефакторингований до наступного:

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        require!(data.data < 100, MyError::DataTooLarge);
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount може зберігати лише дані, менші за 100")]
    DataTooLarge
}
```

# Лабораторна робота

Давайте попрактикуємо концепції, які ми розглянули в цьому уроці, спираючись на програму "Movie Review" з попередніх уроків.

У цій лабораторній роботі ми оновимо програму, щоб карбувати токени користувачам, коли вони надсилають нові рецензії на фільми.

### 1. Початок

Для початку ми використаємо кінцевий стан програми Anchor Movie Review з попереднього уроку. Отже, якщо ви щойно закінчили той урок, то ви все налаштували і готові до роботи. Якщо ж ви тільки починаєте, не хвилюйтеся, ви можете [завантажити стартовий код](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-pdas). Ми будемо використовувати гілку `solution-pdas` як відправну точку.

### 2. Додавання залежностей до `Cargo.toml`

Перш ніж ми почнемо, нам потрібно увімкнути функцію `init-if-needed` і додати контейнер `anchor-spl` до залежностей у `Cargo.toml`. Якщо ви хочете дізнатися більше про функцію `init-if-needed`, перегляньте урок [Прив'язка PDA та облікових записів](anchor-pdas).

```rust
[dependencies]
anchor-lang = { version = "0.25.0", features = ["init-if-needed"] }
anchor-spl = "0.25.0"
```

### 3. Ініціалізуйте токен винагороди

Далі перейдіть до `lib.rs` і створіть інструкцію для ініціалізації карбування нових токенів. Це буде токен, який буде карбуватися кожного разу, коли користувач залишає відгук. Зверніть увагу, що нам не потрібно включати ніякої спеціальної логіки інструкції, оскільки ініціалізація може бути повністю оброблена за допомогою обмежень Anchor.

```rust
pub fn initialize_token_mint(_ctx: Context<InitializeMint>) -> Result<()> {
    msg!("Token mint initialized");
    Ok(())
}
```

Тепер реалізуйте контекстний тип `InitializeMint` і перелічіть облікові записи та обмеження, яких вимагає інструкція. Тут ми ініціалізуємо новий обліковий запис `Mint` за допомогою PDA з рядком "mint" у якості початкового даних. Зауважте, що ми можемо використовувати той самий PDA як для адреси облікового запису `Mint`, так і для органу карбування. Використання PDA в якості органу карбування дозволяє нашій програмі підписуватися на карбування токенів.

Для того, щоб ініціалізувати обліковий запис `Mint`, нам потрібно додати до списку облікових записів `token_program`, `rent` та `ystem_program`.

```rust
#[derive(Accounts)]
pub struct InitializeMint<'info> {
    #[account(
        init,
        seeds = ["mint".as_bytes()],
        bump,
        payer = user,
        mint::decimals = 6,
        mint::authority = mint,
    )]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
    pub system_program: Program<'info, System>
}
```

Вище можуть бути деякі обмеження, які ви ще не бачили. Додавання `mint::decimals` і `mint::authority` разом з `init` гарантує, що обліковий запис буде ініціалізовано як карбування нового токену з відповідним набором десяткових знаків і повноважень.

### 4. Anchor Error

Далі, давайте створимо Anchor Error, який ми будемо використовувати при перевірці значення `rating`, переданого в інструкції `add_movie_review` або `update_movie_review`.

```rust
#[error_code]
enum MovieReviewError {
    #[msg("Rating must be between 1 and 5")]
    InvalidRating
}
```

### 5. Оновлення інструкції `add_movie_review`

Тепер, коли ми виконали деякі налаштування, давайте оновимо інструкцію `add_movie_review` і тип контексту `AddMovieReview`, щоб карбувати токени для рецензента.

Далі, оновіть тип контексту `AddMovieReview`, щоб додати наступні акаунти:

- `token_program` - ми будемо використовувати Token Program для карбування токенів
- `mint` - обліковий запис для карбування токенів, які ми будемо карбувати користувачам, коли вони додаватимуть рецензію на фільм
- `token_account` - асоційований акаунт токенів для вищезгаданого `mint` та рецензента
- `associated_token_program` - необхідний, оскільки ми будемо використовувати обмеження `associated_token` на `token_account`.
- `rent` - обов'язковий, оскільки ми використовуємо обмеження `init-if-needed` на `token_account`

```rust
#[derive(Accounts)]
#[instruction(title: String, description: String)]
pub struct AddMovieReview<'info> {
    #[account(
        init,
        seeds=[title.as_bytes(), initializer.key().as_ref()],
        bump,
        payer = initializer,
        space = 8 + 32 + 1 + 4 + title.len() + 4 + description.len()
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
    // ДОДАЛИ АКАУНТИ НИЖЧЕ
    pub token_program: Program<'info, Token>,
    #[account(
        seeds = ["mint".as_bytes()]
        bump,
        mut
    )]
    pub mint: Account<'info, Mint>,
    #[account(
        init_if_needed,
        payer = initializer,
        associated_token::mint = mint,
        associated_token::authority = initializer
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub rent: Sysvar<'info, Rent>
}
```

Знову ж таки, деякі з наведених вище обмежень можуть бути вам незнайомі. Обмеження `associated_token::mint` і `associated_token::authority` разом з обмеженням `init_if_needed` гарантують, що якщо обліковий запис ще не було ініціалізовано, він буде ініціалізований як асоційований обліковий запис токенів для карбування та повноважень.

Далі, давайте оновимо інструкцію `add_movie_review` наступним чином:

- Перевірте, що `rating` є дійсним. Якщо це не дійсний рейтинг, поверніть помилку `InvalidRating`.
- Створить CPI до інструкції `mint_to` програми токена, використовуючи PDA з повноваженнями карбування в якості підписувача. Зауважте, що ми викарбуємо 10 токенів для користувача, але нам потрібно врахувати десяткові знаки монетного двору, зробивши `10*10^6`.

На щастя, ми можемо використати ящик `anchor_spl` для доступу до допоміжних функцій і типів, таких як `mint_to` і `MintTo` для побудови нашого CPI до програми токенів. Функція `mint_to` приймає в якості аргументів `CpiContext` і ціле число, де ціле число представляє кількість токенів для карбування. `MintTo` можна використовувати для списку облікових записів, які потрібні для інструкції карбування.

```rust
pub fn add_movie_review(ctx: Context<AddMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account created");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.reviewer = ctx.accounts.initializer.key();
    movie_review.title = title;
    movie_review.description = description;
    movie_review.rating = rating;

    mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                authority: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                mint: ctx.accounts.mint.to_account_info()
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint").unwrap()]
            ]]
        ),
        10*10^6
    )?;

    msg!("Minted tokens");

    Ok(())
}
```

### 6. Оновити інструкцію `update_movie_review`

Тут ми лише додаємо перевірку того, що `rating` є дійсним.

```rust
pub fn update_movie_review(ctx: Context<UpdateMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account space reallocated");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.description = description;
    movie_review.rating = rating;

    Ok(())
}
```

### 7. Тест

Це всі зміни, які нам потрібно внести до програми! Тепер давайте оновимо наші тести.

Почніть з того, що переконайтеся, що ваш імпорт над функцією `describe` має такий вигляд:

```typescript
import * as anchor from "@project-serum/anchor"
import { Program } from "@project-serum/anchor"
import { expect } from "chai"
import { getAssociatedTokenAddress, getAccount } from "@solana/spl-token"
import { AnchorMovieReviewProgram } from "../target/types/anchor_movie_review_program"

describe("anchor-movie-review-program", () => {
  // Налаштуйте клієнт на використання локального кластера.
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace
    .AnchorMovieReviewProgram as Program<AnchorMovieReviewProgram>

  const movie = {
    title: "Just a test movie.",
    description: "Wow what a good movie it was real great",
    rating: 5,
  }

  const [movie_pda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from(movie.title), provider.wallet.publicKey.toBuffer()],
    program.programId
  )

  const [mint] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("mint")],
    program.programId
  )
...
}
```

Після цього додайте тест для інструкції `initializeTokenMint`:

```typescript
it("Initializes the reward token", async () => {
    const tx = await program.methods.initializeTokenMint().rpc()
})
```

Зверніть увагу, що нам не довелося додавати `.accounts`, тому що вони будуть виведені, включно з обліковим записом `mint` (якщо у вас увімкнено виведення на основі початкових даних).

Далі, оновіть тест для інструкції `addMovieReview`. Основні доповнення такі:
1. Отримання пов'язаної адреси токена, яку потрібно передати в інструкцію як обліковий запис, що не може бути виведений
2. Перевірте в кінці тесту, що асоційований обліковий запис токена має 10 токенів

```typescript
it("Movie review is added`", async () => {
  const tokenAccount = await getAssociatedTokenAddress(
    mint,
    provider.wallet.publicKey
  )
  
  const tx = await program.methods
    .addMovieReview(movie.title, movie.description, movie.rating)
    .accounts({
      tokenAccount: tokenAccount,
    })
    .rpc()
  
  const account = await program.account.movieAccountState.fetch(movie_pda)
  expect(movie.title === account.title)
  expect(movie.rating === account.rating)
  expect(movie.description === account.description)
  expect(account.reviewer === provider.wallet.publicKey)

  const userAta = await getAccount(provider.connection, tokenAccount)
  expect(Number(userAta.amount)).to.equal((10 * 10) ^ 6)
})
```

Після цього ні тест для `updateMovieReview`, ні тест для `deleteMovieReview` не потребують змін.

На цьому етапі запустіть `anchor test` і ви побачите наступний результат

```console
anchor-movie-review-program
    ✔ Initializes the reward token (458ms)
    ✔ Movie review is added (410ms)
    ✔ Movie review is updated (402ms)
    ✔ Deletes a movie review (405ms)

  5 passing (2s)
```

Якщо вам потрібно більше часу, щоб розібратися з поняттями з цього уроку, або ви застрягли на цьому шляху, не соромтеся поглянути на [код розв'язку](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-add-tokens). Зауважте, що розв'язок до цієї лабораторної роботи знаходиться на гілці `solution-add-tokens`.

# Виклик

Щоб застосувати те, що ви дізналися про CPI в цьому уроці, подумайте, як ви можете включити їх в програму Student Intro. Ви можете зробити щось подібне до того, що ми робили в лабораторній роботі, і додати деякі функції для видачі токенів користувачам, коли вони представляються.

Спробуйте зробити це самостійно, якщо зможете! Але якщо ви застрягли, не соромтеся звертатися до цього [коду рішення](https://github.com/Unboxed-Software/anchor-student-intro-program/tree/cpi-challenge). Зверніть увагу, що ваш код може дещо відрізнятися від коду рішення в залежності від вашої реалізації.


## Завершили завдання?

Завантажте свій код на GitHub і [розкажіть нам, що ви думаєте про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=21375c76-b6f1-4fb6-8cc1-9ef151bc5b0a)!
