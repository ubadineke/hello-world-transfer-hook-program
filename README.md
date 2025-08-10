# Hello World Transfer Hook

This example demonstrates how to implement a basic transfer hook using the SPL Token 2022 Transfer Hook interface to log a simple "Hello Transfer Hook!" message during token transfers.

In this example, every token transfer will trigger the transfer hook to print a greeting message, providing a foundation for understanding how transfer hooks work and can be extended for more complex functionality.

---

## Let's walk through the architecture:

For this program, we will have 1 main state account:

- An ExtraAccountMetaList account

An ExtraAccountMetaList account consists of:

```rust
// This account is managed by the SPL Transfer Hook interface
// and contains metadata about extra accounts required for the transfer hook
```

### In this state account, we will store:

- Extra account metadata: Information about any additional accounts that need to be passed to the transfer hook during execution.
- In this simple example, we don't require any extra accounts, so the list is empty.

This account uses the SPL Transfer Hook interface's standard structure for managing transfer hook metadata.

---

### The system will need to initialize extra account metadata for the transfer hook:

```rust
#[derive(Accounts)]
pub struct InitializeExtraAccountMetaList<'info> {
    #[account(mut)]
    payer: Signer<'info>,

    /// CHECK: ExtraAccountMetaList Account, must use these seeds
    #[account(
        mut,
        seeds = [b"extra-account-metas", mint.key().as_ref()],
        bump
    )]
    pub extra_account_meta_list: AccountInfo<'info>,
    pub mint: InterfaceAccount<'info, Mint>,
    pub token_program: Interface<'info, TokenInterface>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
}
```

Let´s have a closer look at the accounts that we are passing in this context:

- payer: Will be the person initializing the transfer hook metadata. He will be a signer of the transaction, and we mark his account as mutable as we will be deducting lamports from this account.

- extra_account_meta_list: Will be the metadata account that we will initialize. We derive this PDA from the byte representation of "extra-account-metas" and the mint's public key.

- mint: The token mint that will have the transfer hook enabled.

- token_program: The SPL Token 2022 program interface.

- associated_token_program: The SPL Associated Token Account program.

- system_program: Program responsible for the initialization of any new account.

### We then implement some functionality for our InitializeExtraAccountMetaList context:

```rust
pub fn initialize_extra_account_meta_list(
    ctx: Context<InitializeExtraAccountMetaList>,
) -> Result<()> {

    // The `addExtraAccountsToInstruction` JS helper function resolving incorrectly
    let account_metas = vec![

    ];

    // calculate account size
    let account_size = ExtraAccountMetaList::size_of(account_metas.len())? as u64;
    // calculate minimum required lamports
    let lamports = Rent::get()?.minimum_balance(account_size as usize);

    let mint = ctx.accounts.mint.key();
    let signer_seeds: &[&[&[u8]]] = &[&[
        b"extra-account-metas",
        &mint.as_ref(),
        &[ctx.bumps.extra_account_meta_list],
    ]];

    // create ExtraAccountMetaList account
    create_account(
        CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            CreateAccount {
                from: ctx.accounts.payer.to_account_info(),
                to: ctx.accounts.extra_account_meta_list.to_account_info(),
            },
        )
        .with_signer(signer_seeds),
        lamports,
        account_size,
        ctx.program_id,
    )?;

    // initialize ExtraAccountMetaList account with extra accounts
    ExtraAccountMetaList::init::<ExecuteInstruction>(
        &mut ctx.accounts.extra_account_meta_list.try_borrow_mut_data()?,
        &account_metas,
    )?;

    msg!("Extra Account Meta List Initt");
    Ok(())
}
```

In here, we create and initialize the ExtraAccountMetaList account that will store metadata about any extra accounts required for the transfer hook. Since this is a simple hello world example, we don't require any extra accounts, so the account_metas vector is empty.

---

### The transfer hook will execute during every token transfer:

```rust
#[derive(Accounts)]
pub struct TransferHook<'info> {
    #[account(
        token::mint = mint,
        token::authority = owner,
    )]
    pub source_token: InterfaceAccount<'info, TokenAccount>,
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(
        token::mint = mint,
    )]
    pub destination_token: InterfaceAccount<'info, TokenAccount>,
    /// CHECK: source token account owner, can be SystemAccount or PDA owned by another program
    pub owner: UncheckedAccount<'info>,
    /// CHECK: ExtraAccountMetaList Account,
    #[account(
        seeds = [b"extra-account-metas", mint.key().as_ref()],
        bump
    )]
    pub extra_account_meta_list: UncheckedAccount<'info>,
}
```

In this context, we are passing all the accounts needed for transfer hook execution:

- source_token: The token account from which tokens are being transferred. We validate that it belongs to the correct mint and is owned by the owner.

- mint: The token mint being transferred.

- destination_token: The token account to which tokens are being transferred. We validate that it belongs to the correct mint.

- owner: The owner of the source token account. This can be a system account or a PDA owned by another program.

- extra_account_meta_list: The metadata account that contains information about extra accounts required for this transfer hook.

### We then implement some functionality for our TransferHook context:

```rust
pub fn transfer_hook(ctx: Context<TransferHook>, amount: u64) -> Result<()> {

    msg!("Hello Transfer Hook!");

    Ok(())
}
```

In this implementation, we simply log a "Hello Transfer Hook!" message to the program logs. This demonstrates the basic structure of a transfer hook and can be extended to include more complex logic such as validation, logging, or custom business rules.

The transfer hook integrates seamlessly with the SPL Token 2022 transfer process, automatically executing during every transfer attempt and providing a foundation for building more sophisticated transfer hook functionality.

---

### We also implement a fallback instruction handler:

```rust
// fallback instruction handler as workaround to anchor instruction discriminator check
pub fn fallback<'info>(
    program_id: &Pubkey,
    accounts: &'info [AccountInfo<'info>],
    data: &[u8],
) -> Result<()> {
    let instruction = TransferHookInstruction::unpack(data)?;

    // match instruction discriminator to transfer hook interface execute instruction
    // token2022 program CPIs this instruction on token transfer
    match instruction {
        TransferHookInstruction::Execute { amount } => {
            let amount_bytes = amount.to_le_bytes();

            // invoke custom transfer hook instruction on our program
            __private::__global::transfer_hook(program_id, accounts, &amount_bytes)
        }
        _ => return Err(ProgramError::InvalidInstructionData.into()),
    }
}
```

This fallback handler is necessary to work around Anchor's instruction discriminator check and properly handle the transfer hook interface's execute instruction that is called by the Token 2022 program during transfers.

---

This hello world transfer hook provides a simple foundation for understanding how transfer hooks work with SPL Token 2022. It demonstrates the basic structure and can be extended to include more complex functionality such as access control, logging, analytics, or custom business logic while maintaining the standard token interface that users and applications expect.
