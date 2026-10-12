use anchor_lang::prelude::*;
use anchor_lang::system_program;

declare_id!("11111111111111111111111111111111");

#[program]
pub mod tip_splitter {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let tip_jar = &mut ctx.accounts.tip_jar;
        tip_jar.authority = ctx.accounts.authority.key();
        tip_jar.bump = ctx.bumps.tip_jar;
        Ok(())
    }

    pub fn tip(ctx: Context<Tip>, amount: u64) -> Result<()> {
        let cpi_context = CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            system_program::Transfer {
                from: ctx.accounts.tipper.to_account_info(),
                to: ctx.accounts.tip_jar.to_account_info(),
            },
        );
        system_program::transfer(cpi_context, amount)?;
        Ok(())
    }

    pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
        **ctx.accounts.tip_jar.to_account_info().try_borrow_mut_lamports()? -= amount;
        **ctx.accounts.authority.to_account_info().try_borrow_mut_lamports()? += amount;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 1,
        seeds = [b"tip_jar", authority.key().as_ref()],
        bump
    )]
    pub tip_jar: Account<'info, TipJar>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Tip<'info> {
    #[account(mut, seeds = [b"tip_jar", tip_jar.authority.as_ref()], bump = tip_jar.bump)]
    pub tip_jar: Account<'info, TipJar>,
    #[account(mut)]
    pub tipper: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Withdraw<'info> {
    #[account(
        mut, 
        seeds = [b"tip_jar", tip_jar.authority.as_ref()], 
        bump = tip_jar.bump, 
        has_one = authority
    )]
    pub tip_jar: Account<'info, TipJar>,
    #[account(mut)]
    pub authority: Signer<'info>,
}

#[account]
pub struct TipJar {
    pub authority: Pubkey,
    pub bump: u8,
}
