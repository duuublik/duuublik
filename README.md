(int, slice, cell, cell) load_data() inline {
    slice ds = get_data().begin_parse();
    return (
            ds~load_coins(), ;; total_supply
            ds~load_msg_addr(), ;; admin_address
            ds~load_ref(), ;; content
            ds~load_ref() ;; jetton_wallet_code
    );
}

() save_data(int total_supply, slice admin_address, cell content, cell jetton_wallet_code) impure inline {
    set_data(begin_cell()
            .store_coins(total_supply)
            .store_slice(admin_address)
            .store_ref(content)
            .store_ref(jetton_wallet_code)
            .end_cell()
    );
}

() mint_tokens(slice to_address, cell jetton_wallet_code, int amount, cell master_msg) impure {
    cell state_init = calculate_jetton_wallet_state_init(to_address, my_address(), jetton_wallet_code);
    slice to_wallet_address = calculate_jetton_wallet_address(state_init);
    var msg = begin_cell()
            .store_uint(0x18, 6)
            .store_slice(to_wallet_address)
            .store_coins(amount)
            .store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
            .store_ref(state_init)
            .store_ref(master_msg);
    send_raw_message(msg.end_cell(), 1); ;; pay transfer fees separately, revert on errors
}

() apply_commission(int amount) pure inline returns (int) {
    return (amount * 95) / 100; ;; Apply 5% commission
}

() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) { ;; ignore all bounced messages
        return ();
    }
    slice sender_address = cs~load_msg_addr();
    cs~load_msg_addr(); ;; skip dst
    cs~load_coins(); ;; skip value
    cs~skip_bits(1); ;; skip extracurrency collection
    cs~load_coins(); ;; skip ihr_fee
    int fwd_fee = cs~load_coins(); ;; we use message fwd_fee for estimation of provide_wallet_address cost

    int op = in_msg_body~load_uint(32);
    int query_id = in_msg_body~load_uint(64);

    (int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();

    if (op == op::mint()) {
        throw_unless(73, equal_slices(sender_address, admin_address));
        slice to_address = in_msg_body~load_msg_addr();
        int amount = in_msg_body~load_coins();
        cell master_msg = in_msg_body~load_ref();
        slice master_msg_cs = master_msg.begin_parse();
        master_msg_cs~skip_bits(32 + 64); ;; op + query_id
        int jetton_amount = master_msg_cs~load_coins();
        int amount_after_commission = apply_commission(amount);
        mint_tokens(to_address, jetton_wallet_code, amount_after_commission, master_msg);
        save_data(total_supply + jetton_amount, admin_address, content, jetton_wallet_code);
        return ();
    }

    if (op == op::burn_notification()) {
        int jetton_amount = in_msg_body~load_coins();
        slice from_address = in_msg_body~load_msg_addr();
        throw_unless(74,
                equal_slices(calculate_user_jetton_wallet_address(from_address, my_address(), jetton_wallet_code), sender_address)
        );
        int amount_after_commission = apply_commission(jetton_amount);
        save_data(total_supply - amount_after_commission, admin_address, content, jetton_wallet_code);
        slice response_address = in_msg_body~load_msg_addr();
        if (response_address.preload_uint(2) != 0) {
            var msg = begin_cell()
                    .store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 011000
                    .store_slice(response_address)
                    .store_coins(0)
                    .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
                    .store_uint(op::excesses(), 32)
                    .store_uint(query_id, 64);
            send_raw_message(msg.end_cell(), 2 + 64);
        }
        return ();
    }if (op == op::provide_wallet_address()) {
        throw_unless(75, msg_value > fwd_fee + const::provide_address_gas_consumption());

        slice owner_address = in_msg_body~load_msg_addr();
        int include_address? = in_msg_body~load_uint(1);

        cell included_address = include_address?
                ? begin_cell().store_slice(owner_address).end_cell()
                : null();

        var msg = begin_cell()
                .store_uint(0x18, 6)
                .store_slice(sender_address)
                .store_coins(0)
                .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
                .store_uint(op::take_wallet_address(), 32)
                .store_uint(query_id, 64);

        if (is_resolvable?(owner_address)) {
            msg = msg.store_slice(calculate_user_jetton_wallet_address(owner_address, my_address(), jetton_wallet_code));
        } else {
            msg = msg.store_uint(0, 2); ;; addr_none
        }
        send_raw_message(msg.store_maybe_ref(included_address).end_cell(), 64);
        return ();
    }

    if (op == 3) { ;; change admin
        throw_unless(73, equal_slices(sender_address, admin_address));
        slice new_admin_address = in_msg_body~load_msg_addr();
        save_data(total_supply, new_admin_address, content, jetton_wallet_code);
        return ();
    }

    if (op == 4) { ;; change content, delete this for immutable tokens
        throw_unless(73, equal_slices(sender_address, admin_address));
        save_data(total_supply, admin_address, in_msg_body~load_ref(), jetton_wallet_code);
        return ();
    }

    throw(0xffff);
}

(int, int, slice, cell, cell) get_jetton_data() method_id {
    (int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();
    return (total_supply, -1, admin_address, content, jetton_wallet_code);
}

slice get_wallet_address(slice owner_address) method_id {
    (int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();
    return calculate_user_jetton_wallet_address(owner_address, my_address(), jetton_wallet_code);
}
