---
назва: Довільний CPI
цілі:
- Пояснити ризики безпеки, пов'язані з викликом CPI до невідомої програми
- Продемонструвати, як модуль Anchor's CPI запобігає цьому під час виконання CPI з однієї програми-якоря до іншої
- Безпечно та надійно виконати CPI з якірної програми до довільної неякірної програми
---

# TL;DR

- Щоб згенерувати CPI, цільова програма повинна бути передана в інструкцію, що викликає, як обліковий запис. Це означає, що в інструкцію можна передати будь-яку цільову програму. Ваша програма повинна перевіряти наявність некоректних або неочікуваних програм.
- Виконуйте перевірку програм у власних програмах, просто порівнюючи відкритий ключ переданої програми з програмою, яку ви очікували.
- Якщо програма написана на Anchor, то вона може мати загальнодоступний модуль CPI. Це робить виклик програми з іншої програми Anchor простим і безпечним. Модуль Anchor CPI автоматично перевіряє, чи збігається адреса переданої програми з адресою програми, що зберігається в модулі.

# Огляд

Міжпрограмний виклик (CPI) - це коли одна програма викликає інструкцію в іншій програмі. "Довільний міжпрограмний виклик" - це коли програма структурована таким чином, що вона видає міжпрограмний виклик будь-якій програмі, яка передається в інструкцію, а не очікує виконання міжпрограмного виклику з однією конкретною програмою. Враховуючи те, що користувачі вашої програми можуть передати будь-яку програму до списку облікових записів, не перевіривши адресу переданої програми, ваша програма виконує CPI для довільних програм.

Відсутність перевірки програми створює можливість для зловмисника ввести іншу програму, ніж очікувалося, і змусити оригінальну програму викликати інструкцію на цій таємничій програмі. Невідомо, якими можуть бути наслідки цього CPI. Це залежить від програмної логіки (як оригінальної програми, так і несподіваної програми), а також від того, які ще облікові записи були передані в оригінальну інструкцію.

## Відсутні перевірки програми

Розглянемо наступну програму як приклад. Інструкція `cpi` викликає інструкцію `transfer` на `token_program`, але немає коду, який би перевіряв, чи обліковий запис `token_program`, переданий в інструкцію, насправді є програмою токенів SPL.

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod arbitrary_cpi_insecure {
    use super::*;

    pub fn cpi(ctx: Context<Cpi>, amount: u64) -> ProgramResult {
        solana_program::program::invoke(
            &spl_token::instruction::transfer(
                ctx.accounts.token_program.key,
                ctx.accounts.source.key,
                ctx.accounts.destination.key,
                ctx.accounts.authority.key,
                &[],
                amount,
            )?,
            &[
                ctx.accounts.source.clone(),
                ctx.accounts.destination.clone(),
                ctx.accounts.authority.clone(),
            ],
        )
    }
}

#[derive(Accounts)]
pub struct Cpi<'info> {
    source: UncheckedAccount<'info>,
    destination: UncheckedAccount<'info>,
    authority: UncheckedAccount<'info>,
    token_program: UncheckedAccount<'info>,
}
```

Зловмисник може легко викликати цю інструкцію і передати дублікат програми-токену, яку він створив і контролює.

## Додавання програмних перевірок

Цю уразливість можна виправити, просто додавши декілька рядків до інструкції `cpi` для перевірки того, чи є відкритий ключ програми `token_program` відкритим ключем програми токенів SPL.

```rust
pub fn cpi_secure(ctx: Context<Cpi>, amount: u64) -> ProgramResult {
    if &spl_token::ID != ctx.accounts.token_program.key {
        return Err(ProgramError::IncorrectProgramId);
    }
    solana_program::program::invoke(
        &spl_token::instruction::transfer(
            ctx.accounts.token_program.key,
            ctx.accounts.source.key,
            ctx.accounts.destination.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?,
        &[
            ctx.accounts.source.clone(),
            ctx.accounts.destination.clone(),
            ctx.accounts.authority.clone(),
        ],
    )
}
```

Тепер, якщо зловмисник передасть іншу програму-токен, інструкція поверне помилку `ProgramError::IncorrectProgramId`.

Залежно від програми, яку ви викликаєте за допомогою CPI, ви можете або жорстко закодувати адресу очікуваного ідентифікатора програми, або використати Rust-упаковку програми для отримання адреси програми, якщо вона доступна. У наведеному вище прикладі, у скриньці `spl_token` міститься адреса програми SPL Token.

## Використання модуля Anchor CPI

Простіший спосіб керування перевірками програми - це використання модулів Anchor CPI. У [попередньому уроці](https://github.com/Unboxed-Software/solana-course/blob/main/content/anchor-cpi) ми дізналися, що Anchor може автоматично генерувати модулі CPI, щоб спростити введення CPI у програму. Ці модулі також підвищують безпеку, перевіряючи відкритий ключ програми, який передається в одну з її відкритих інструкцій.

Кожна програма Anchor використовує макрос `declare_id()` для визначення адреси програми. Коли модуль CPI генерується для певної програми, він використовує адресу, передану у цей макрос, як "джерело істини" і автоматично перевіряє, що всі CPI, зроблені за допомогою модуля CPI, націлені на цей ідентифікатор програми.

Хоча по суті це нічим не відрізняється від ручної перевірки програми, використання модулів CPI дозволяє уникнути можливості забути виконати перевірку програми або випадково ввести неправильний ідентифікатор програми під час її жорсткого кодування.

У наведеній нижче програмі показано приклад використання модуля CPI для програми SPL Token для виконання передачі, показаної у попередніх прикладах.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod arbitrary_cpi_recommended {
    use super::*;

    pub fn cpi(ctx: Context<Cpi>, amount: u64) -> ProgramResult {
        token::transfer(ctx.accounts.transfer_ctx(), amount)
    }
}

#[derive(Accounts)]
pub struct Cpi<'info> {
    source: Account<'info, TokenAccount>,
    destination: Account<'info, TokenAccount>,
    authority: Signer<'info>,
    token_program: Program<'info, Token>,
}

impl<'info> Cpi<'info> {
    pub fn transfer_ctx(&self) -> CpiContext<'_, '_, '_, 'info, token::Transfer<'info>> {
        let program = self.token_program.to_account_info();
        let accounts = token::Transfer {
            from: self.source.to_account_info(),
            to: self.destination.to_account_info(),
            authority: self.authority.to_account_info(),
        };
        CpiContext::new(program, accounts)
    }
}
```

Зверніть увагу, що, як і в наведеному вище прикладі, Anchor створив кілька [обгорток для популярних нативних програм](https://github.com/coral-xyz/anchor/tree/master/spl/src), які дозволяють вам видавати CPI в них так, ніби вони є програмами Anchor.

Крім того, залежно від програми, до якої ви робите CPI, ви можете використовувати тип облікового запису Anchor [`Program`](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/program/struct.Program.html) для валідації переданої програми у вашій структурі валідації облікового запису. Між ящиками [`anchor_lang`](https://docs.rs/anchor-lang/latest/anchor_lang) та [`anchor_spl`](https://docs.rs/anchor_spl/latest/), наступні типи `Program` надаються з коробки:

- [`System`](https://docs.rs/anchor-lang/latest/anchor_lang/struct.System.html)
- [`AssociatedToken`](https://docs.rs/anchor-spl/latest/anchor_spl/associated_token/struct.AssociatedToken.html)
- [`Token`](https://docs.rs/anchor-spl/latest/anchor_spl/token/struct.Token.html)

Якщо у вас є доступ до модуля CPI якірної програми, ви зазвичай можете імпортувати її тип програми наступним чином, замінивши назву програми на назву самої програми:

```rust
use other_program::program::OtherProgram;
```

# Лабораторна робота

Щоб показати важливість звірки з програмою, яку ви використовуєте для CPI, ми попрацюємо зі спрощеною і дещо надуманою грою. Ця гра представляє персонажів з обліковими записами КПК і використовує окрему програму "метаданих" для керування метаданими персонажів та їхніми атрибутами, такими як здоров'я та сила.

Хоча цей приклад дещо надуманий, насправді архітектура майже ідентична тому, як працюють NFT на Solana: програма SPL Token Program керує карбуванням, розповсюдженням та передачею токенів, а окрема програма метаданих використовується для присвоєння метаданих токенам. Таким чином, вразливість, яку ми розглядаємо тут, може бути застосована і до реальних токенів.

### 1. Налагодження

Ми почнемо з гілки `starter` [цього сховища](https://github.com/Unboxed-Software/solana-arbitrary-cpi/tree/starter). Клонуйте сховище, а потім відкрийте його у гілці `starter`.

Зверніть увагу, що там є три програми:

1. `gameplay`
2. `character-metadata`
3. `fake-metadata`

Крім того, у каталозі `tests` вже є тест.

Перша програма, `gameplay`, є тією, яку безпосередньо використовує наш тест. Погляньте на програму. Вона містить дві інструкції:

1. `create_character_insecure` - створює новий символ і CPI у програмі метаданих для налаштування початкових атрибутів символу
2. `battle_insecure` - зіштовхує двох персонажів один проти одного, присвоюючи "перемогу" персонажу з найвищими атрибутами

Друга програма, `character-metadata`, призначена для обробки метаданих символів. Погляньте на цю програму. Вона містить єдину інструкцію `create_metadata`, яка створює новий КПК і призначає псевдовипадкове значення від 0 до 20 для здоров'я та сили персонажа.

Остання програма, `fake-metadata` - це програма "фальшивих" метаданих, призначена для ілюстрації того, що зловмисник може зробити, щоб використати нашу програму `gameplay`. Ця програма майже ідентична до програми `character-metadata`, тільки вона встановлює початковий рівень здоров'я та потужності персонажа на максимально допустимому рівні: 255.

### 2. Протестувати інструкцію `create_character_insecure`

У каталозі `tests` вже є тест для цього. Він довгий, але витратьте хвилину на його перегляд, перш ніж ми обговоримо його разом:

```typescript
it("Insecure instructions allow attacker to win every time", async () => {
    // Ініціалізуйте програвач один з реальними метаданими програми
    await gameplayProgram.methods
      .createCharacterInsecure()
      .accounts({
        metadataProgram: metadataProgram.programId,
        authority: playerOne.publicKey,
      })
      .signers([playerOne])
      .rpc()

    // Ініціалізація зловмисника за допомогою програми з підробленими метаданими
    await gameplayProgram.methods
      .createCharacterInsecure()
      .accounts({
        metadataProgram: fakeMetadataProgram.programId,
        authority: attacker.publicKey,
      })
      .signers([attacker])
      .rpc()

    // Отримати обидва акаунти з метаданими гравця
    const [playerOneMetadataKey] = getMetadataKey(
      playerOne.publicKey,
      gameplayProgram.programId,
      metadataProgram.programId
    )

    const [attackerMetadataKey] = getMetadataKey(
      attacker.publicKey,
      gameplayProgram.programId,
      fakeMetadataProgram.programId
    )

    const playerOneMetadata = await metadataProgram.account.metadata.fetch(
      playerOneMetadataKey
    )

    const attackerMetadata = await fakeMetadataProgram.account.metadata.fetch(
      attackerMetadataKey
    )

    // Звичайний гравець повинен мати здоров'я та силу в межах від 0 до 20
    expect(playerOneMetadata.health).to.be.lessThan(20)
    expect(playerOneMetadata.power).to.be.lessThan(20)

    // Нападник матиме здоров'я та силу 255
    expect(attackerMetadata.health).to.equal(255)
    expect(attackerMetadata.power).to.equal(255)
})
```

У цьому тесті розглядається сценарій, в якому звичайний гравець і зловмисник створюють своїх персонажів. Тільки зловмисник передає ідентифікатор програми підробленої програми метаданих, а не власне програми метаданих. А оскільки інструкція `create_character_insecure` не має жодних програмних перевірок, вона все одно виконується.

В результаті звичайний персонаж має відповідну кількість здоров'я і сили: кожна з них має значення від 0 до 20. Але здоров'я і сила зловмисника дорівнюють 255, що робить його непереможним.

Якщо ви цього ще не зробили, запустіть `anchor test`, щоб переконатися, що цей тест дійсно поводиться так, як описано.

### 3. Створіть інструкцію `create_character_secure` для створення символів

Давайте виправимо це, створивши безпечну інструкцію для створення нового символу. Ця інструкція повинна реалізовувати належні програмні перевірки і використовувати для обчислення ІСЦ контейнер `character-metadata` програми `cpi`, а не просто використовувати `invoke`.

Якщо ви хочете перевірити свої навички, спробуйте зробити це самостійно, перш ніж рухатися далі.

Ми почнемо з оновлення нашого оператора `use` у верхній частині файлу `gameplay` programs `lib.rs`. Ми надамо собі доступ до типу програми для перевірки акаунта та допоміжної функції для видачі ІСЦ `create_metadata`.

```rust
use character_metadata::{
    cpi::accounts::CreateMetadata,
    cpi::create_metadata,
    program::CharacterMetadata,
};
```

Далі створимо нову структуру валідації облікового запису під назвою `CreateCharacterSecure`. Цього разу ми зробимо `metadata_program` типом `Program`:

```rust
#[derive(Accounts)]
pub struct CreateCharacterSecure<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 64,
        seeds = [authority.key().as_ref()],
        bump
    )]
    pub character: Account<'info, Character>,
    #[account(
        mut,
        seeds = [character.key().as_ref()],
        seeds::program = metadata_program.key(),
        bump,
    )]
    /// CHECK: ручні перевірки
    pub metadata_account: AccountInfo<'info>,
    pub metadata_program: Program<'info, CharacterMetadata>,
    pub system_program: Program<'info, System>,
}
```

Нарешті, ми додаємо інструкцію `create_character_secure`. Вона буде такою ж, як і раніше, але використовуватиме повну функціональність Anchor CPI, а не використовуватиме безпосередньо `invoke`:

```rust
pub fn create_character_secure(ctx: Context<CreateCharacterSecure>) -> Result<()> {
    let character = &mut ctx.accounts.character;
    character.metadata = ctx.accounts.metadata_account.key();
    character.auth = ctx.accounts.authority.key();
    character.wins = 0;

    let context = CpiContext::new(
        ctx.accounts.metadata_program.to_account_info(),
        CreateMetadata {
            character: ctx.accounts.character.to_account_info(),
            metadata: ctx.accounts.metadata_account.to_owned(),
            authority: ctx.accounts.authority.to_account_info(),
            system_program: ctx.accounts.system_program.to_account_info(),
        },
    );

    create_metadata(context)?;

    Ok(())
}
```

### 4. Тест `create_character_secure` (створити_символ_безпечно)

Тепер, коли у нас є безпечний спосіб ініціалізації нового символу, давайте створимо новий тест. Цей тест повинен просто спробувати ініціалізувати символ зловмисника і очікувати на помилку.

```typescript
it("Secure character creation doesn't allow fake program", async () => {
    try {
      await gameplayProgram.methods
        .createCharacterSecure()
        .accounts({
          metadataProgram: fakeMetadataProgram.programId,
          authority: attacker.publicKey,
        })
        .signers([attacker])
        .rpc()
    } catch (error) {
      expect(error)
      console.log(error)
    }
})
```

Запустіть "якірний тест", якщо ви цього ще не зробили. Зверніть увагу, що, як і очікувалося, було видано помилку, яка вказує на те, що ідентифікатор програми, переданий в інструкцію, не є очікуваним ідентифікатором програми:

```bash
'Program log: AnchorError caused by account: metadata_program. Error Code: InvalidProgramId. Error Number: 3008. Error Message: Program ID was not as expected.',
'Program log: Left:',
'Program log: FKBWhshzcQa29cCyaXc1vfkZ5U985gD5YsqfCzJYUBr',
'Program log: Right:',
'Program log: D4hPnYEsAx4u3EQMrKEXsY3MkfLndXbBKTEYTwwm25TE'
```

Це все, що вам потрібно зробити, щоб захиститися від довільних ІСЦ!

Бувають випадки, коли вам потрібна більша гнучкість у використанні CPI у вашій програмі. Ми, звичайно, не будемо заважати вам створювати програму так, як вам потрібно, але, будь ласка, вживайте всіх можливих заходів обережності, щоб не допустити вразливостей у вашій програмі.

Якщо ви хочете поглянути на остаточний код рішення, ви можете знайти його на гілці `solution` у [цьому ж репозиторії](https://github.com/Unboxed-Software/solana-arbitrary-cpi/tree/solution).

# Виклик

Як і в інших уроках цього розділу, ваша можливість попрактикуватися в уникненні цієї вразливості полягає в аудиті власних або інших програм.

Витратьте трохи часу, щоб переглянути хоча б одну програму і переконатися, що перевірка програм виконується для кожної програми, що передається в інструкції, особливо для тих, які викликаються через CPI.

Пам'ятайте, якщо ви знайшли помилку або уразливість у чужій програмі, будь ласка, повідомте про це! Якщо ви знайшли таку помилку у власній програмі, не забудьте негайно виправити її.

## Виконали завдання?

Завантажте свій код на GitHub і [розкажіть нам, що ви думаєте про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=5bcaf062-c356-4b58-80a0-12cca99c29b0)!
