# Token Fundraiser

This example demonstrates how to stake a NFT using the Token Metadata Program.

In this example, a user will be able to stake a NFT of a certain collection, get points, and mint rewards.

---

## Let's walk through the architecture:

For this program, we will have 3 state accounts:

- A User account:

- A Stake Config account:

- A Stake account

A User account consists of:

```rust
#[account]
pub struct UserAccount {
    pub points: u32,
    pub amount_staked: u8,
    pub bump: u8,
}

impl Space for UserAccount {
    const INIT_SPACE: usize = 8 + 4 + 1 + 1;
}
```

### In this state account, we will store:

- points: the current points that the user has

- amount_staked: to number of NFTs that the user has currently staked

- bump: since our User account will be a PDA (Program Derived Address), we will store the bump of the account

We use InitSpace derive macro to implement the space triat that will calculate the amount of space that our account will use on-chain (without taking the anchor discriminator into consideration)

---

A Stake Config account consists of:

```rust
#[account]
pub struct StakeConfig {
    pub points_per_stake: u8,
    pub max_stake: u8,
    pub freeze_period: u32,
    pub rewards_bump: u8,
    pub bump: u8,
}

impl Space for StakeConfig {
    const INIT_SPACE: usize = 8 + 1 + 1 + 4 + 1 + 1;
}
```

### In this state account, we will store:

- points_per_stake: the points that the user will receive per stake

- freeze_period: to time that the NFT need to be staked before being unstaked

- rewards_bump: we will be initializing a rewards mint based on a PDA (Program Derived Address), so we will store that PDA bump

- bump: since our Stake Config account will be a PDA, we will store the bump of the account

Again, we use InitSpace derive macro to implement the space triat that will calculate the amount of space that our account will use on-chain (without taking the anchor discriminator into consideration)

---

A Stake account consists of:

```rust
#[account]
pub struct StakeAccount {
    pub owner: Pubkey,
    pub mint: Pubkey,
    pub collection: Pubkey,
    pub last_update: i64,
    pub bump: u8,
}

impl Space for StakeAccount {
    const INIT_SPACE: usize = 8 + 32 + 32 + 32 + 8 + 1;
}
```

### In this state account, we will store:

- owner: the owner of this stake account

- mint: to addrss of the NFT staked

- collection: the collection of the staked NFT

- last_update: the time the NFT was staked

- bump: since our Stake account will be a PDA, we will store the bump of the account

Again, we use InitSpace derive macro to implement the space triat that will calculate the amount of space that our account will use on-chain (without taking the anchor discriminator into consideration)

---

### The user will be able to create new User accounts. For that, we create the following context:

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    #[account(
        init,
        payer = user,
        seeds = [b"user".as_ref(), user.key().as_ref()],
        bump,
        space = UserAccount::INIT_SPACE,
    )]
    pub user_account: Account<'info, UserAccount>,
    pub system_program: Program<'info, System>,
}
```

Let´s have a closer look at the accounts that we are passing in this context:

- user: will be the person starting the user account. He will be a signer of the transaction, and we mark his account as mutable as we will be deducting lamports from this account

- user_account: will be the state account that we will initialize and the user will be paying for the initialization of the account.
We derive the User PDA from the byte representation of the word "user" and the reference of the user publick key. Anchor will calculate the canonical bump (the first bump that throws that address out of the ed25519 eliptic curve) and save it for us in a struct

- system_program: Program resposible for the initialization of any new account

### We then implement some functionality for our Initialize context:

```rust
impl<'info> Initialize<'info> {
    
    pub fn initialize_user(&mut self, bumps: &InitializeBumps) -> Result<()> {
        self.user_account.set_inner(UserAccount { 
            points: 0, 
            amount_staked: 0, 
            bump: bumps.user_account 
        });

        Ok(())
    }
}
```

In here, we basically just set the initial data of our User account

---

### The admin of the platform will be able to create Config accounts:

```rust
#[derive(Accounts)]
pub struct InitializeConfig<'info> {
    #[account(mut)]
    pub admin: Signer<'info>,
    #[account(
        init, 
        payer = admin,
        seeds = [b"config".as_ref(), collection_mint.key().as_ref()],
        bump,
        space = StakeConfig::INIT_SPACE,
    )]
    pub config: Account<'info, StakeConfig>,
    pub collection_mint: Account<'info, Mint>,
    #[account(
        init,
        payer = admin,
        seeds = [b"rewards".as_ref(), config.key().as_ref()],
        bump,
        mint::decimals = 6,
        mint::authority = config,
    )]
    pub rewards_mint: Account<'info, Mint>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
}
```

In this context, we are passing all the accounts needed to contribute to a fundraising campaign:

- Admin: the address of the platform admin. He will be a signer of the transaction, and we mark his account as mutable as we will be deducting lamports from this account

- collection_mint: the address of the collection NFT that is targeted for this config

- config: will be the state account that we will initialize and the admin will be paying for the initialization of the account.
We derive the User PDA from the byte representation of the word "config" and the reference of the collection NFT address. Anchor will calculate the canonical bump (the first bump that throws that address out of the ed25519 eliptic curve) and save it for us in a struct

- rewards_mint: We initialize a rewards mint account that will be used to mint rewards to the user. We initialize it with 6 decimals, and the authority of that mint account will be the config account (so that our program can mint tokens)

- system_program: Program resposible for the initialization of any new account

- token_program: We are initializing a Mint account

### We then implement some functionality for our Initialize Config context:

```rust
impl<'info> InitializeConfig<'info> {
    pub fn initialize_config(&mut self, points_per_stake: u8, max_stake: u8, freeze_period: u32, bumps: &InitializeConfigBumps) -> Result<()> {
        self.config.set_inner(StakeConfig {
            points_per_stake,
            max_stake,
            freeze_period,
            rewards_bump: bumps.rewards_mint,
            bump: bumps.config,
        });

        Ok(())
    }
}
```
In here, we basically just set the initial data of our Stake Config account

---

### The user will be able to stake NFTs:

```rust
#[derive(Accounts)]
pub struct Stake<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub mint: Account<'info, Mint>,
    pub collection_mint: Account<'info, Mint>,
    #[account(
        mut,
        associated_token::mint = mint,
        associated_token::authority = user,
    )]
    pub mint_ata: Account<'info, TokenAccount>,
    #[account(
        seeds = [
            b"metadata",
            metadata_program.key().as_ref(),
            mint.key().as_ref()
        ],
        seeds::program = metadata_program.key(),
        bump,
        constraint = metadata.collection.as_ref().unwrap().key.as_ref() == collection_mint.key().as_ref(),
        constraint = metadata.collection.as_ref().unwrap().verified == true,
    )]
    pub metadata: Account<'info, MetadataAccount>,
    #[account(
        seeds = [
            b"metadata",
            metadata_program.key().as_ref(),
            mint.key().as_ref(),
            b"edition"
        ],
        seeds::program = metadata_program.key(),
        bump,
    )]
    pub edition: Account<'info, MasterEditionAccount>,
    #[account(
        seeds = [b"config".as_ref(), collection_mint.key().as_ref()],
        bump = config.bump,
    )]
    pub config: Account<'info, StakeConfig>,
    #[account(
        init,
        payer = user,
        space = StakeAccount::INIT_SPACE,
        seeds = [b"stake".as_ref(), mint.key().as_ref(), config.key().as_ref()],
        bump,
    )]
    pub stake_account: Account<'info, StakeAccount>,
    #[account(
        mut,
        seeds = [b"user".as_ref(), user.key().as_ref()],
        bump = user_account.bump,
    )]
    pub user_account: Account<'info, UserAccount>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub metadata_program: Program<'info, Metadata>,
}
```

In this context, we are passing all the accounts needed for a user to stake his NFT:

- user: The address of the person staking the NFT. We mark it as mutable since the user will be paying for initialization fees.

- mint: the NFT that the user is staking.

- collection_mint: the Collection NFT to which the NFT being staked belongs to.

- metadata: The metadata account of the NFT. 
In here, we check that the this metadata account belongs to the NFT being staked.
We also check that the NFT belongs to the correct collection and that is verified as part of that collection.

- edition: The master_edition account of the NFT.

- config: The Stake Config account being used to stake this NFT.

- stake_account: will be the state account that we will initialize and the user will be paying for the initialization of the account.
We derive the User PDA from the byte representation of the word "stake", the reference of the NFT address and the reference of the Config account address. Anchor will calculate the canonical bump (the first bump that throws that address out of the ed25519 eliptic curve) and save it for us in a struct

### We then implement some functionality for our Stake context:

```rust
impl<'info> Stake<'info> {
    pub fn stake(&mut self, bumps: &StakeBumps) -> Result<()> {

        require!(self.user_account.amount_staked < self.config.max_stake, StakeError::MaxStakeReached);

        self.stake_account.set_inner(StakeAccount {
            owner: self.user.key(),
            mint: self.mint.key(),
            collection: self.collection_mint.key(),
            last_update: Clock::get()?.unix_timestamp,
            bump: bumps.stake_account,
        });

        let cpi_program = self.token_program.to_account_info();

        let cpi_accounts = Approve {
            to: self.mint_ata.to_account_info(),
            delegate: self.stake_account.to_account_info(),
            authority: self.user.to_account_info(),
        };

        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        
        approve(cpi_ctx, 1)?;

        let seeds = &[
            b"stake",
            self.mint.to_account_info().key.as_ref(),
            self.config.to_account_info().key.as_ref(),
            &[self.stake_account.bump]
        ];     
        let signer_seeds = &[&seeds[..]];

        let delegate = &self.stake_account.to_account_info();
        let token_account = &self.mint_ata.to_account_info();
        let edition = &self.edition.to_account_info();
        let mint = &self.mint.to_account_info();
        let token_program = &self.token_program.to_account_info();
        let metadata_program = &self.metadata_program.to_account_info();

        FreezeDelegatedAccountCpi::new(
            metadata_program,
            FreezeDelegatedAccountCpiAccounts {
                delegate,
                token_account,
                edition,
                mint,
                token_program,
            },
        ).invoke_signed(signer_seeds)?;

        self.user_account.amount_staked += 1;

        Ok(())
    }
}
```

In this implementation, we start by setting the initial state of the Stake account.

After that, we delegate authority over the NFT to the stake account and freeze the NFT in the user's wallet.

Finally, we increment the total staked amount in the User account.

---

### Users will be able to unstake their contributions, if the freeze period has been elapsed:

```rust
#[derive(Accounts)]
pub struct Unstake<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub mint: Account<'info, Mint>,
    #[account(
        mut,
        associated_token::mint = mint,
        associated_token::authority = user,
    )]
    pub mint_ata: Account<'info, TokenAccount>,
    #[account(
        seeds = [
            b"metadata",
            metadata_program.key().as_ref(),
            mint.key().as_ref(),
            b"edition"
        ],
        seeds::program = metadata_program.key(),
        bump,
    )]
    pub edition: Account<'info, MasterEditionAccount>,
    pub config: Account<'info, StakeConfig>,
    #[account(
        mut,
        close = user,
        seeds = [b"stake".as_ref(), mint.key().as_ref(), config.key().as_ref()],
        bump,
    )]
    pub stake_account: Account<'info, StakeAccount>,
    #[account(
        mut,
        seeds = [b"user".as_ref(), user.key().as_ref()],
        bump = user_account.bump,
    )]
    pub user_account: Account<'info, UserAccount>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub metadata_program: Program<'info, Metadata>,
}
```

In this context, we are passing all the accounts needed for a contributor to unstake his NFT:

- user: The address of the user that wants to unstake the NFT

- mint: The NFT being unstaked.

- mint_ata: The user ATA where the NFT is frozen

- edition: The NFT master edition account

- config: An initialized Stake Config account that will ne used to perform checks and award points

- stake_account: The NFT Stake Account. We will be closing this account and sending the rent back to the user.

- user_account: The User account where will be updating the total amount staked and the total points.

- system_account: We will be closing accounts.

- token_program: We will performing CPIs (Cross Program Invocations) to the token program to revoke authority over the NFT.

- metadata_program: We will be performing CPIs to the token metadata program to thaw / unfreeze the NFT

### We then implement some functionality for our Unstake context:

```rust
impl<'info> Unstake<'info> {
    pub fn unstake(&mut self) -> Result<()> {

        let time_elapsed = ((Clock::get()?.unix_timestamp - self.stake_account.last_update) / 86400) as u32;

        require!(time_elapsed >= self.config.freeze_period, StakeError::FreezePeriodNotPassed);

        self.user_account.points += time_elapsed as u32 * self.config.points_per_stake as u32;

        let seeds = &[
            b"stake",
            self.mint.to_account_info().key.as_ref(),
            self.config.to_account_info().key.as_ref(),
            &[self.stake_account.bump]
        ];     
        let signer_seeds = &[&seeds[..]];

        let delegate = &self.stake_account.to_account_info();
        let token_account = &self.mint_ata.to_account_info();
        let edition = &self.edition.to_account_info();
        let mint = &self.mint.to_account_info();
        let token_program = &self.token_program.to_account_info();
        let metadata_program = &self.metadata_program.to_account_info();
        
        ThawDelegatedAccountCpi::new(
            metadata_program,
            ThawDelegatedAccountCpiAccounts {
                delegate,
                token_account,
                edition,
                mint,
                token_program,
            }
        ).invoke_signed(signer_seeds)?;

        let cpi_program = self.token_program.to_account_info();

        let cpi_accounts = Revoke {
            source: self.mint_ata.to_account_info(),
            authority: self.user.to_account_info(),
        };

        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);

        revoke(cpi_ctx)?;

        self.user_account.amount_staked -= 1;

        Ok(())
    }
}
```

In here, we will check if the freeze period has been elapsed and throw an error if it didn't.

After that, we will update the user points based on the time stsked and the points per stake from the config account.

We then thaw the delegated account and revoke authority over the NFT.

Finally, we update the total amount staked in the User account

---

### Users will be able to claim their rewards:

```rust
#[derive(Accounts)]
pub struct Claim<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub collection_mint: Account<'info, Mint>,
    #[account(
        mut,
        seeds = [b"user".as_ref(), user.key().as_ref()],
        bump = user_account.bump,
    )]
    pub user_account: Account<'info, UserAccount>,
    #[account(
        mut,
        seeds = [b"rewards".as_ref(), config.key().as_ref()],
        bump = config.rewards_bump
    )]
    pub rewards_mint: Account<'info, Mint>,
    #[account(
        seeds = [b"config".as_ref(), collection_mint.key().as_ref()],
        bump = config.bump,
    )]
    pub config: Account<'info, StakeConfig>,
    #[account(
        init_if_needed,
        payer = user,
        associated_token::mint = rewards_mint,
        associated_token::authority = user,
    )]
    pub rewards_ata: Account<'info, TokenAccount>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}
```

In this context, we are passing all the accounts needed for a contributor to unstake his NFT:

### We then implement some functionality for our Claim context:

```rust
impl<'info> Claim<'info> {
    pub fn claim(&mut self) -> Result<()> {

        let cpi_program = self.token_program.to_account_info();

        let seeds = &[
            b"config".as_ref(),
            self.collection_mint.to_account_info().key.as_ref(),
            &[self.config.bump]
        ];     
        let signer_seeds = &[&seeds[..]];

        let cpi_accounts = MintTo {
            mint: self.rewards_mint.to_account_info(),
            to: self.rewards_ata.to_account_info(),
            authority: self.config.to_account_info(),
        };

        let cpi_context = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer_seeds);

        mint_to(cpi_context, self.user_account.points as u64 * 10_u64.pow(self.rewards_mint.decimals as u32))?;

        self.user_account.points = 0;
        
        Ok(())
    }
}
```

In here, we will mint the rewards to the user.

Finally, we reset his total points to 0.