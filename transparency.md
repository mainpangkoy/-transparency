//= @transparency <ITEM_ID>
//= Wait until you get the results in your chat
//= To add more tables (like other storages) check arrays .@tables$ and .@id$
//= F_GET_REAL_OWNER_NAME(<"string table">,<id>) returns IGN / username label
//============================================================
function	script	F_GET_REAL_OWNER_NAME	{
	.@table$ = getarg(0);
	.@id     = getarg(1);
	.@n$     = "";

	if (.@table$ != "account_id" && .@table$ != "char_id") {
		if (.@table$ != "id")  return "(Mails):";
		else                   return "(Guilds):";
	}

	.@acc = .@id;

	if (.@table$ == "char_id") {
		query_sql("SELECT `account_id`,`name` FROM `char` WHERE `char_id` = '" + .@id + "'", .@acc, .@charname$);
		if (.@charname$ != "") return .@charname$; // found IGN
		// else fall through to account username
	}

	query_sql("SELECT `group_id`,`userid` FROM `login` WHERE `account_id` = '" + .@acc + "'", .@g, .@name$);
	if (.@g >= 89)      .@n$ = "[GM]";
	else if (.@g >= 1)  .@n$ = "[CM]";
	return .@n$ + .@name$;
}

-	script	AnalyzeItem	-1,{
OnInit:
	bindatcmd("transparency",        strnpcinfo(3)+"::OnCommand2",  0, 99);
end;

OnCommand2:
	.@owner_name = true; 
OnCommand:
	.@item_id = atoi(.@atcmd_parameters$[0]);
	if (.@item_id <= 0) {
		dispbottom "@transparency <ITEM_ID>", 0xFFFFFF;
		end;
	}
	if (getitemname(.@item_id) == "null") {
		dispbottom "Invalid Item ID!", 0xFFFFFF;
		dispbottom "@transparency <ITEM_ID>", 0xFFFFFF;
		end;
	}

	freeloop(1);

	setarray .@tables$, "cart_inventory", "guild_storage", "inventory", "storage", "mail_attachments";
	setarray .@id$,     "char_id",        "guild_id",      "char_id",   "account_id","id";

	dispbottom "Searching for item '" + getitemname(.@item_id) + "'", 0xFFFFFF;
	dispbottom "Analyze Tables", 0xFFFFFF;

	.@MAX_REF = 30;

	.@allcounts = 0;

	for (.@i = 0; .@i < getarraysize(.@tables$); .@i++) {
		query_sql(
			"SELECT `amount`,`" + .@id$[.@i] + "`, COALESCE(`refine`,0) AS `refine` "
			+ "FROM `" + .@tables$[.@i] + "` "
			+ "WHERE `nameid` = '" + .@item_id + "'",
			.@amount, .@owner_id, .@refine
		);

		for (.@n = 0; .@n < getarraysize(.@amount); .@n++) {
			.@amt = .@amount[.@n];
			.@allcounts += .@amt;

			if (.@refine[.@n] >= 0) {
				.@refine_count[ .@refine[.@n] ] += .@amt;
			}

			if (.@owner_name) {
				.@owner$ = F_GET_REAL_OWNER_NAME(.@id$[.@i], .@owner_id[.@n]);
				.@ndx = inarray(.@name_list$, .@owner$);
				if (.@ndx == -1) {
					.@ndx = getarraysize(.@name_list$);
					.@name_list$[.@ndx] = .@owner$;
				}
				.@count_list[.@ndx] += .@amt;

				.@r = .@refine[.@n];
				if (.@r >= 0 && .@r < .@MAX_REF) {
					.@key = .@ndx * .@MAX_REF + .@r;
					.@owner_refine_count[ .@key ] += .@amt;
				}

				sleep2 2;
			}
		}

		sleep2 5;
		deletearray .@amount[0],   getarraysize(.@amount);
		deletearray .@owner_id[0], getarraysize(.@owner_id);
		deletearray .@refine[0],   getarraysize(.@refine);
	}

	if (.@owner_name) {
		dispbottom "==================================", 0xFFFFFF;
		dispbottom "Extended List (Owner / IGN with Refine):", 0xFFFFFF;

		for (.@i = 0; .@i < getarraysize(.@name_list$); .@i++) {
			.@line$ = .@name_list$[.@i] + " : ";
			.@first = 1;

			for (.@r = 0; .@r < .@MAX_REF; .@r++) {
				.@key = .@i * .@MAX_REF + .@r;
				.@cnt = .@owner_refine_count[ .@key ];
				if (.@cnt > 0) {
					if (!.@first) .@line$ += ", ";
					.@line$ += "+" + .@r + " x" + .@cnt;
					.@first = 0;
				}
			}

			if (.@first) .@line$ += "none"; // (just in case)
			dispbottom .@line$, 0xFFFFFF;
		}
		dispbottom "==================================", 0xFFFFFF;
	}

	dispbottom "Refine Breakdown (Global):", 0xFFFFFF;
	for (.@r = 0; .@r < getarraysize(.@refine_count); .@r++) {
		if (.@refine_count[.@r] > 0) {
			dispbottom "+" + .@r + " : " + .@refine_count[.@r], 0xFFFFFF;
		}
	}

	dispbottom "Analyze Done.", 0xFFFFFF;
	dispbottom "(" + .@allcounts + ") " + getitemname(.@item_id) + ".", 0xFFFFFF;

	freeloop(0);
end;
}
