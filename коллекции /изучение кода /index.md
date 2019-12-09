RandomSeed(iSeed + 1);
            
flRandPi_1 = RandomFloat(0, 0x40C90FDBu);
flCosRandPi_1 = cos(flRandPi_1);
flSinRandPi_1 = sin(flRandPi_1);
fRandInaccuracy = RandomFloat(0, flInaccuracy);
v48 = flCosRandPi_1 * fRandInaccuracy;
v47 = fRandInaccuracy * flSinRandPi_1;
            
while ( iBullet < FileWeaponInfo_t::GetAttributeInt(pWeaponInfo, "bullets", 0) )
{
flRandPi_2 = RandomFloat(0, 0x40C90FDBu);
flSinRandPi_2 = sin(flRandPi_2);
flCosRandPi_2 = cos(flRandPi_2);
flRandSpread = RandomFloat(0, LODWORD(flSpread));
v54[iBullet] = flCosRandPi_2 * flRandSpread;
v55[iBullet++] = flRandSpread * flSinRandPi_2;
}

while ( iBullet2 < FileWeaponInfo_t::GetAttributeInt(pWeaponInfo, "bullets", 0) )
{
flSpreadY = v47 + v55[iBullet2];
v32 = v48 + v54[iBullet2++];
flSpreadX = v32;
...
}
and by following into C_CSPlayer::FireBullet( ..., flSpreadX, flSpreadY ) with this:
AngleVectors(shootAngles0, &vecDirShooting, &vecRight, &vecUp);
...
vecDir.x = (float)((float)(vecRight.x * flSpreadX) + vecDirShooting.x) + (float)(vecUp.x * flSpreadY);
vecDir.y = (float)((float)(flSpreadX * vecRight.y) + vecDirShooting.y) + (float)(flSpreadY * vecUp.y);
vecDir.z = (float)((float)(vecRight.z * flSpreadX) + vecDirShooting.z) + (float)(vecUp.z * flSpreadY);
So where do you get flSpread and flInaccuracy? By looking where FX_FireBullets is called you will find C_WeaponCSBaseGun::CSBaseGunFire and see this:

C_WeaponCSBaseGun::CSBaseGunFire<al>(int a1<ecx>, ... )
{
    pWeapon = a1;
    ...
     (*(void (__thiscall **)(int))(*(_DWORD *)pWeapon + 1840))(pWeapon);// GetSpread()
     *(float *)&flSpread = v7;
     (*(void (__thiscall **)(int))(*(_DWORD *)pWeapon + 1836))(pWeapon);// GetInaccuracy()
    *(float *)&flInaccuracy = v7;
}


So now we put this all together:
#define WEAPON_SPREAD_OFFSET (1836/4)

float CNoSpread::GetInaccuracy( C_BaseCombatWeapon *pWeapon )
{
    DWORD dwInaccuracyVMT = ( *reinterpret_cast< PDWORD_PTR* >( pWeapon ) )[ WEAPON_SPREAD_OFFSET ];
    return ( ( float ( __thiscall* )( C_BaseCombatWeapon* ) ) dwInaccuracyVMT )( pWeapon );
}

float CNoSpread::GetSpread( C_BaseCombatWeapon *pWeapon )
{
    DWORD dwSpreadVMT = ( *reinterpret_cast< PDWORD_PTR* >( pWeapon ) )[ WEAPON_SPREAD_OFFSET + 1 ];
    return ( ( float ( __thiscall* )( C_BaseCombatWeapon* ) ) dwSpreadVMT )( pWeapon );
}

void CNoSpread::UpdateAccuracyPenalty( C_BaseCombatWeapon *pWeapon )
{
    DWORD dwUpdateVMT = ( *reinterpret_cast< PDWORD_PTR* >( pWeapon ) )[ WEAPON_SPREAD_OFFSET + 2 ];
    return ( ( void ( __thiscall* )( C_BaseCombatWeapon* ) ) dwUpdateVMT )( pWeapon );
}

void CNoSpread::CompansateSpread( CUserCmd *pCmd, C_BaseCombatWeapon *pWeapon )
{
    Vector vecForward, vecRight, vecUp, vecAntiDir;
    float flSpread, flInaccuracy, flSpreadX, flSpreadY;
    QAngle qAntiSpread;

    UpdateAccuracyPenalty( pWeapon );

    flSpread = GetSpread( pWeapon );
    flInaccuracy = GetInaccuracy( pWeapon );

    RandomSeed( (pCmd->random_seed & 0xFF) + 1 );

    float flRandPi_1 = RandomFloat( 0.0f, 2.0f  *  M_PI_F );
    float flRandInaccuracy = RandomFloat( 0.0f, flInaccuracy );
    float flRandPi_2 = RandomFloat( 0.0f, 2.0f  * M_PI_F );
    float flRandSpread = RandomFloat( 0.0f, flSpread );

    float flSpreadX = cos(flRandPi_1) * flRandInaccuracy + cos(flRandPi_2) * flRandSpread;
    float flSpreadY = sin(flRandPi_1) * flRandInaccuracy + sin(flRandPi_2) * flRandSpread;

    AngleVectors( pCmd->viewangles, &vecForward, &vecRight, &vecUp );

    vecAntiDir = vecDirShooting + ( vecRight * -flSpreadX ) + ( vecUp * -flSpreadY );
    vecAntiDir.NormalizeInPlace( );

    VectorAngles( vecAntiDir, qAntiSpread );

    pCmd->viewangles = qAntiSpread;

}
